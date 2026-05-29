# Writing upstream bug reports that land with OSS maintainers

Notes from filing a thorough, well-received bug report on a Dark Reader memory leak ([darkreader/darkreader#14164 comment](https://github.com/darkreader/darkreader/issues/14164#issuecomment-4569799075)). The principles below are general; the specific Dark Reader analysis lives in [chrome-extension-spa-leaks.md](chrome-extension-spa-leaks.md).

## Decide first: new issue or comment on existing?

If an open issue already covers the same bug class — even if it was reported against a different site — **comment on it**. Don't fragment the thread. A thorough comment on an existing issue:

- Builds on conversation history the maintainer has already engaged with
- Lets you address prior pushback the original reporter couldn't (e.g., "I couldn't reproduce in a clean profile")
- Shows multiple sites are affected, which matters for prioritization

File a new issue only when nothing on the tracker matches at all, or when the existing issue is clearly about a different mechanism.

## Lead with what the maintainer asked for before

Read the existing thread before writing. If the maintainer asked the original reporter for something they couldn't provide (clean profile, single extension, exact repro steps, headless capture), open with **that**. Don't litigate the earlier response. Just provide what was missing.

Example opener that worked:
> Hi @maintainer — adding clean-profile reproducer + source-code trace for this leak. I ran into the same issue on `[reproducer site]` and worked through it with heap snapshots, analysis assistance from Claude Code, and a read of `[relevant source dir]`.
>
> Short version: I am able to reliably reproduce the issue in a clean profile with only [extension] installed.

This sets up "I'm here to add evidence, not argue."

## Credit other reporters whose observations went unfollowed

If someone else in the thread had a partial answer or correct lead the maintainer didn't engage with, name them by @-handle and connect their observation to yours. Two effects:

- They feel validated and may join the thread to confirm
- The maintainer sees independent corroboration
- You look like you actually read the thread

## Present evidence, let the reader conclude

The most common failure mode in bug reports is **telling the maintainer what their bug is** before showing them why. They'll bristle. Instead:

- Use hedges like "appears to be," "would match," "what you'd expect if" for causal claims
- Use direct factual claims for code semantics ("a `.then` callback is retained until the upstream Promise settles") — those are not opinions about the codebase
- Use direct factual claims for observed evidence ("the four functions grew by +10,534 to +10,539 — all four within 5 of each other") — those are measurements

Example:
- ❌ "The leak is in the async image modifier's interaction with `imageSelectorQueue`."
- ✓ "The dominant leak appears to be in the async image modifier's interaction with `imageSelectorQueue` — which would match `@otherperson`'s comment about CSS Cache references."

By the end of the report the reader should arrive at the same conclusion you have, but through the evidence rather than your assertion.

## Don't pre-judge their costs

Avoid telling the maintainer what something will cost them in their codebase. Phrases to cut:

- "At the cost of introducing a new pattern in this codebase"
- "Lowest review friction"
- "Smallest delta in idiom"
- "Should be easy to land"
- "Won't affect performance"

You don't know their review process, their internal priorities, what other features are in flight, or what they consider "easy." State factual properties of each option and let them decide what the cost is.

## When suggesting fixes, lead with what matches their existing patterns

If you propose multiple shapes, the order matters. Lead with the option that fits existing idioms in their codebase, then offer the more ambitious alternative. This reads as "I noticed your conventions" rather than "here's how I'd write it."

Don't propose more than two or three shapes. More than that looks like you're asking them to architect for you. And drop genuinely awkward options entirely — including a third option just to look thorough dilutes the others.

## Frame fix offers as an afterthought, not the centerpiece

The body of the report is the analysis. The fix offer is a closer. Structure:

1. Repro
2. Evidence (heap signatures, measurements, file:line references)
3. Code trace through the bug
4. Secondary observations
5. Tooling you used (if relevant — link a public gist of any analyzer scripts)
6. **A possible fix** — one or two shapes with neutral tradeoff notes
7. **Conditional PR offer** — "happy to put a PR together in either shape if you'd find one useful"

The fix sketch should feel like "here's where I'd start if you want me to" — not "I propose this be merged."

## Acknowledge AI assistance honestly, don't hide it or center it

If you used Claude Code, ChatGPT, etc. as part of the investigation, name it but frame it as a tool:

> "...worked through it with heap snapshots, analysis assistance from Claude Code, and a read of `[source dir]`."

Don't say "Claude Code found..." (sounds AI-generated). Don't say only "I found..." (dishonest, and the maintainer will sometimes spot the style anyway). Naming the specific tool is better than vague "AI assistance" — Claude Code is a known Anthropic CLI, not some random chatbot.

The maintainer's stance on AI-assisted dev is unknown. The honest middle reduces the risk of being dismissed as "just copy-pasted from a chat window."

## Don't make claims you can't substantiate

Common traps:

- **Timing claims from file mtimes.** DevTools heap snapshots' mtimes are *save* times (when you clicked Save), not capture times. The interval between filenames is meaningless. Don't quote rates ("grew at 200 MB/min") unless you actually timed it with a stopwatch.
- **"This obviously is..."** — overconfident language that sets up a fight if the maintainer disagrees with one detail. State the evidence, let it speak.
- **"It should be simple to fix"** — you don't know their constraints.
- **"It's not a [vendor] bug"** — true if you can show it, but if you can show it then the data shows it without you saying so. Save the assertion for your own internal notes.

## Don't frame the report site as "the thing that should be fixed"

If you reproduced on a third party's product (GCP Console, Grafana docs, etc.), name the site as the **reproducer**, not the **target**. Otherwise the maintainer might dismiss it as "well, take it up with [vendor]." The bug is in the extension; the site just makes it visible.

## Sanitize snapshots before sharing publicly

Heap snapshots are JSON containing string fragments from the page DOM and JS heap. Expect to find:

- Your signed-in email (from auth UI)
- Account display name
- Project IDs, OAuth client IDs, internal API paths
- Sometimes cached API response data

Sanitize with `sed` before sharing. Heap snapshots index strings by array position, so length-varying replacements inside string values don't break the format. See [heap-snapshot-streaming-analysis.md](heap-snapshot-streaming-analysis.md) for the technique.

After sanitizing, verify by re-parsing the file through a streaming reader — if it produces sensible output, the JSON is intact.

## Make the linked artifacts low-friction for the maintainer

- **Shared folder, not zip file.** Browsers will refuse to download multi-GB zips, and uncompressing GB of heap snapshots is a chore.
- **Public Drive link, "anyone with the link, viewer."** Maintainers may not want to sign in to a Google account they don't use.
- **Test the link in an Incognito window** (or have someone else hit it) before posting. Broken links waste reply cycles.
- **Include a README** in the share folder explaining what each file is and how to use it (e.g., "load both into DevTools' Memory tab, switch to Comparison view").
- **If you wrote analyzer scripts**, publish them as a public gist and link to that — saves the maintainer from having to recreate your tooling.

## Conditional PR offer at the end

Don't promise a PR without confirmation. Don't ask "what direction should I take?" — that puts the work on the maintainer. The right shape:

> "Happy to put a PR together in either shape if you'd find one useful — the [test patch] already builds clean on commit `[hash]`, and extending it with [the next piece] is straightforward from there."

Notice:
- Conditional on their interest
- Doesn't presuppose acceptance
- Shows you've already done the build/test groundwork
- Names the smallest next unit of work, not a vague "I'll fix it"

## After posting

GitHub auto-subscribes you to issues you comment on. You don't need to do anything to get reply notifications. If the maintainer engages and picks a direction, the fork you've already set up locally has the test patch ready to extend.

If the issue stays quiet for weeks, **don't bump it.** The comment itself is a useful reference for the next person who searches for the symptom — searches for the specific function names (`imageSelectorQueue`, `addListener` lockstep, etc.) will surface it. That alone is worth the writeup.

## What to skip

Several things are tempting but counterproductive:

- ❌ **Apologizing for length.** If the report is long because the bug is complex, the length is doing real work.
- ❌ **"I hope this is helpful."** Just submit the work.
- ❌ **Speculating about the maintainer's motives** ("you probably haven't had time to look at this, but...").
- ❌ **Ratings/severity tags they didn't ask for.** Most repos don't use them; some do but want maintainers to assign them.
- ❌ **A summary of what you'll cover.** Just cover it.

## Related

- [chrome-extension-spa-leaks.md](chrome-extension-spa-leaks.md) — the case study this was developed against
- [heap-snapshot-streaming-analysis.md](heap-snapshot-streaming-analysis.md) — analysis tooling referenced from upstream reports
