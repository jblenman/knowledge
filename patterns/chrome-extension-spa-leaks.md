# Chrome extension memory leaks on SPAs

When a single Chrome tab grows to multiple gigabytes on a complex SPA (Material Design + Angular apps especially — GCP Console, Firebase Console, Grafana docs, Material-based admin panels), check for style-overriding extensions **before** blaming the site.

## Dark Reader signature (verified 2026-05-26 on GCP Credentials page, clean profile)

Symptom: tab on `console.cloud.google.com` → APIs & Services → Credentials grows from a few hundred MB to 10+ GB during a session. Pins 200-500% CPU even when idle (heap is large enough that GC walks dominate). Page becomes unresponsive; on a 16 GB machine the renderer is killed by macOS jetsam. **The leak doesn't require user interaction — loading the page is sufficient.**

What it looks like in heap snapshots:

| constructor | growth across captures | meaning |
|---|---|---|
| `var(--darkreader-bg--*)`, `var(--darkreader-background-*)`, etc. | from ~0 to thousands of distinct strings | Dark Reader's CSS override custom properties |
| `addListener`, `callback`, `getDeclarations`, `removeListeners` | grow in **identical lockstep** (within 5-8 of each other) | Dark Reader's per-CSS-variable hook quartet from `variables.ts:getModifierForVariable` |
| `CSSStyleRule`, `CSSStyleDeclaration`, `CSSRuleList`, `StylePropertyMap`, `CSSMediaRule` | **20-80× growth** | Dark Reader retains references to every style rule it overrides |
| `InternalNode` (DOM internal nodes) | 70-80× | downstream of CSS processing |
| `Object`, `(concatenated string)` | 20-80× | objects backing the above |
| RxJS `Subscription`, `Observable` | **1.1-1.4× (flat)** | the SPA's own code is *not* leaking |

The single most telling sign: the four functions `addListener`, `callback`, `getDeclarations`, `removeListeners` grow by *exactly the same number* in lockstep across captures. That's mechanical — only an automated per-thing registration pattern produces four named functions in perfect parallel.

## Root cause (two distinct leak paths in `src/inject/dynamic-theme/`)

Source-code analysis traced the leak to two paths, only the first of which is fixable with the simpler "missing cleanup" fix:

### Path A — `StyleManager.destroy()` doesn't clean up its `sheetModifier`

- `createStyleSheetModifier()` (`stylesheet-modifier.ts:62`) holds a `varTypeChangeCleaners: Set<() => void>` in its closure.
- Each `handleVarDeclarations()` call (`stylesheet-modifier.ts:255`) registers a per-CSS-variable callback on the **singleton** `variablesStore.typeChangeSubscriptions` Map (`variables.ts:42`), and adds a corresponding cleaner to the Set.
- Cleaners drain at the start of every `modifySheet()` call (`stylesheet-modifier.ts:226-227`) — fine while the manager is alive.
- But `StyleManager.destroy()` (`style-manager.ts:495-505`) never tells `sheetModifier` to clean up. When the parent `<style>` element is removed from the DOM and `removeManager()` (`index.ts:467-471`) fires, the listener callbacks stay registered in `typeChangeSubscriptions` forever, retaining `rebuildVarRule` → the entire `modifySheet` closure → CSS rule objects.

Patched locally and verified: adding a `destroy()` method to `StyleSheetModifier` that drains the cleaners + clears caches, called from `StyleManager.destroy()` and from the two `AdoptedStyleSheetManager` variants, makes the listener quartet stay flat across captures. **But it doesn't meaningfully reduce overall growth.**

### Path B — async-modifier closure retention via `imageSelectorQueue` (dominant)

- `modify-css.ts:514` parks `Promise` resolvers indefinitely in the module-level `imageSelectorQueue` Map whenever a CSS selector isn't currently in the DOM.
- The async `getBgImageModifier` function (`modify-css.ts:500`) awaits these parked Promises. The async function never returns; `modified` stays pending.
- `handleAsyncDeclaration` (`stylesheet-modifier.ts:241`) attaches a `.then` callback to `modified`. That `.then` callback's closure captures `rebuildAsyncRule`, which lives in the same `modifySheet` lexical environment as `rebuildVarRule`, `varDeclarations` Map, `asyncDeclarations` Map, etc.
- A `.then` callback is retained as long as the upstream Promise hasn't settled. If a selector never appears in the DOM (think `:hover`, `:focus`, `.mat-dialog-overlay`, `.mat-tooltip-shown`, scope-attribute selectors for components that don't mount in a given session), the resolver — and the whole `modifySheet` closure — stays alive forever.

This accounts for the bulk of the CSS-object growth.

## Why Material/Angular pages amplify this

Material Design and Angular's emulated view encapsulation synthesize CSS rules at component-mount time for a wide range of state-dependent selectors users may never trigger in a given session. Every rule containing a `background-image` parks a Promise in `imageSelectorQueue`. Across hundreds of components × dozens of state selectors × repeated re-rendering, the queue accumulates pending resolvers and each one pins a `modifySheet` closure.

Pages with mostly static CSS don't trigger it nearly as hard.

## Fix

**For users:** per-site disable in Dark Reader (extension popup → site list) instantly stops the leak on that origin. No need to uninstall.

**For upstream:** the gap in `StyleManager.destroy()` and adopted-style-manager destroy paths is filled by a `StyleSheetModifier.destroy()` method that drains `varTypeChangeCleaners`. The `imageSelectorQueue` path requires plumbing cancellation through the modifier API (an `AbortSignal` from `createStyleSheetModifier()` raced against the parked Promise, or a per-modifier `onCancel` callback list that matches Dark Reader's existing `cleaners` pattern in `adopted-style-manger.ts`).

Filed at [darkreader/darkreader#14164](https://github.com/darkreader/darkreader/issues/14164) on 2026-05-28 with clean-profile heap snapshots, full Path A/Path B trace, and the fix sketches above. Comment URL: https://github.com/darkreader/darkreader/issues/14164#issuecomment-4569799075.

## Quick triage when this signature shows up

```bash
# Verify Dark Reader is installed
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ \
  | xargs -I{} sh -c 'cat ~/Library/Application\ Support/Google/Chrome/Default/Extensions/{}/*/manifest.json 2>/dev/null \
    | python3 -c "import json,sys; m=json.load(sys.stdin); print(\"{} ->\", m.get(\"name\"))"'
```

Dark Reader's typical extension ID is `eimadpbcbfnmbkopoojfekhnkhdbieeh`. If present and the heap-snapshot signature above matches, it's this bug.

## Other style-overriding extensions with similar risk profiles

- **Stylus** — user CSS injection, can have similar issues if rules are heavy
- **Midnight Lizard** — another dark-mode forcer
- **Stylish** (deprecated but some users still have it)
- Any privacy-blocker that injects CSS rules (most modern ad blockers use `declarativeNetRequest` API and don't show this pattern)

## Don't file with the SPA vendor first

If you see this signature, the SPA's own code is innocent. File with the extension. Dark Reader is open-source at https://github.com/darkreader/darkreader — they take heap-snapshot reports seriously, but expect the report to address Alexander Shutau's standard "I couldn't reproduce in a clean profile" pushback by providing exactly that. See `oss-maintainer-bug-reports.md` for what makes these reports land.

## Sanitizing snapshots before sharing

Heap snapshots are JSON and will contain string fragments from the DOM and page — your email (from auth UI), GCP project IDs, OAuth client IDs, etc. Sanitize with `sed` before sharing publicly:

```bash
# Heap snapshots index strings by *array position*, not byte offset,
# so length-varying replacements inside string values don't corrupt the format.
sed -f scrub.sed input.heapsnapshot > sanitized.heapsnapshot
```

Where `scrub.sed` contains line-oriented substitutions like:
```
s/your-email@example\.com/redacted@example.com/g
s/your-project-id/dr-leak-repro-proj1/g
s/123456789012/PROJECT-NUMBER/g
```

Validate after by re-parsing through `heap_stream_count.py` (the streaming analyzer) — if it produces sensible output, the JSON is still valid.

## Related

- [heap-snapshot-streaming-analysis.md](heap-snapshot-streaming-analysis.md) — how to analyze multi-GB heap snapshots that DevTools can't load
- [oss-maintainer-bug-reports.md](oss-maintainer-bug-reports.md) — communication patterns for upstream bug reports on OSS projects
