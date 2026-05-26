# Evaluating LLM Prompt Guidance — The Three-Layer Model

A reusable framework for deciding which prompt advice transfers across models, which is locked to a specific model's training, and which is timeless craft. Useful when porting prompts between vendors, deciding which old patterns to keep when a new model lands, and explaining to teams why "what worked last year" might not work now.

## The three layers

| Layer | What it is | Transferability | Lifespan |
|---|---|---|---|
| **Model-intrinsic** | Behavior baked into a specific model's training, or features of a specific product | None — applies only to the named model/product | Tied to the model version |
| **General LLM physics** | Properties of how transformers and tool-using agents work | High — applies to any modern LLM | Stable for years |
| **Craft / philosophical** | Patterns that work given current model capability; reflects the gap between what the model does naturally and what you want | Variable — works on capable enough models, may be necessary or harmful depending | Shifts as models improve |

Prompt guidance documents typically mix all three. Recognizing which layer a piece of advice lives in is the key skill.

## Layer 1: Model-intrinsic

These are baked-in behaviors or product features. Copying them across models will mislead you.

Recognition patterns:

- The advice references a specific model name or version
- The advice references a specific parameter, API endpoint, or product feature
- The advice describes an idiosyncratic behavior ("this model stops abruptly if you do X")

Examples from the 2026 landscape:

- Codex CLI's "no upfront plans during rollouts" — literally trained behavior of GPT-5.3-Codex / GPT-5.5 in Codex CLI; Opus doesn't do it
- Specific edit-format preferences (`apply_patch` for Codex, others for Claude) — each model is trained on a particular diff format
- Claude Opus 4.7 removed manual `budget_tokens` (returns 400 error) — API change, not a craft choice
- The `phase` parameter in OpenAI's Responses API — product feature, not a general principle
- Self-aware context window introspection — only some models were trained with it
- Cost / billing artifacts (e.g., GPT-5.5's 272K input pricing cliff) — vendor-specific
- Harness features (`<system-reminder>` injection, AGENTS.md auto-discovery, `/btw` overlay) — product features

Rule: when guidance includes a model name, version, parameter name, or product feature, treat it as model-intrinsic until proven otherwise.

## Layer 2: General LLM physics

These derive from how transformers actually work, so they apply across vendors and tend to be stable across model generations.

Recognition patterns:

- The advice ties to a property of how transformer attention or generation works
- The advice describes a universal LLM failure mode
- The advice is about token economics or context-window behavior

Examples:

- **Lost-in-the-middle attention degradation** — transformers attend most strongly to the start and end of context, weaker in the middle. Universal.
- **Performance degrades as context fills** — true on every long-context model. Affects both Anthropic and OpenAI lineups in similar ways.
- **Prompt caching layout** (static content first, volatile content last) — function of how cache key matching works, not a model preference.
- **Verification > trust** — hedge against the universal LLM property of producing plausible-but-wrong output. Both Anthropic and OpenAI call this their highest-leverage practice.
- **Tight feedback loops beat long ones** — correcting fast and clearing context are both more effective than long-running corrections.
- **Specific beats vague** — concrete examples and references almost always outperform abstract instructions.
- **Long sessions accumulate noise** — true for any agent that retains conversation history.

Rule: physics-derived advice transfers cleanly across modern LLMs. Worth carrying forward when porting prompts.

## Layer 3: Craft / philosophical

This is the interesting category — neither hard-trained nor pure physics, but reflecting what works given current model capability. The line moves over time as models improve.

Recognition patterns:

- The advice tells the model HOW to think (vs WHAT to deliver)
- The advice prescribes a process the model should follow
- The advice could plausibly be wrong on a sufficiently capable model

Examples and how the line has moved:

- **"Describe outcomes, not process"** — Increasingly true as models improve. In 2022 you had to spell out chain-of-thought; in 2024 you had to ask for it; in 2026 capable models do it natively, and asking explicitly competes for tokens with actual context.
- **"Don't over-coach reasoning"** — Only became true once models got good at reasoning natively. The "consider 2 alternatives, think step-by-step" advice that helped GPT-3.5 and Claude 1 is now actively harmful on GPT-5.5.
- **Modular system prompt structure (Role / Goal / Success / Constraints / Output / Stop rules)** — Pure ergonomics. Works on any modern model; helps you write clearer prompts whether the model strictly needs the structure.
- **"Avoid ALWAYS/NEVER absolutes"** — Behavior-dependent. Some models will literally satisfy "commit and push everything" by force-overriding `.gitignore`. Others read it as guidance and ask. The same advice has different *necessity* depending on the model.
- **"Use plan mode before implementing"** — Universal craft (think before doing) but model-intrinsic for implementation (Claude Code's plan mode is built in; Codex CLI's prompt guide says don't prompt for plans).

Rule: craft advice usually applies but check the line. If a pattern was right for the previous model generation and feels wrong on the new one, the line has probably moved.

## The migration heuristic

When carrying prompt patterns across models, ask in order:

1. **Does the guidance reference a model name, version, parameter, or product feature?** → Model-intrinsic. Don't port it.
2. **Does it derive from "transformers attend more to recent + early tokens" or "LLMs hallucinate plausibly"?** → Universal. Port it.
3. **Does it encode a specific behavior gap ("tell the model to X because it doesn't do X by default")?** → Test on the new model. If the new model already does X natively, drop the instruction. If it doesn't, keep it but check whether it's still net-positive.

## The "shifting line" insight

Craft advice migrates between categories as models improve:

| Era | "Think step-by-step" instruction status |
|---|---|
| 2022 (GPT-3.5) | **Necessary** — without it, the model didn't reason at all |
| 2024 (GPT-4 / Claude 3) | **Helpful** — marginal lift |
| Early 2026 (GPT-5.4, Opus 4.5) | **Neutral** — model already does it |
| Mid 2026 (GPT-5.5, Opus 4.7) | **Actively harmful on GPT-5.5; redundant on Opus** |

Same words, same underlying principle, four different categories in four years. This is why old AGENTS.md and CLAUDE.md files accumulate cruft — they encode gaps that closed as models got better, and the obsolete coaching starts costing more than it pays.

## Convergence vs divergence in the 2026 landscape

Anthropic and OpenAI's prompt guidance has converged on most points:

- Both favor short, outcome-first prompts
- Both warn against over-coaching reasoning
- Both moved to adaptive reasoning (Anthropic's adaptive thinking ≈ OpenAI's reasoning_effort)
- Both emphasize verification as the highest-leverage practice
- Both warn that more reasoning ≠ better

Where they diverge — and these are the model-intrinsic differences worth knowing:

- **Plan / preamble behavior** — Claude Code's explore-then-implement is built into the harness and recommended; Codex CLI explicitly warns against prompting for upfront plans (model trained to stop early)
- **Absolute rules** — Claude Code is more comfortable with "IMPORTANT" / "YOU MUST" emphasis; OpenAI's guidance discourages absolutes for judgment calls
- **Verbosity** — Claude defaults to fairly chatty; GPT-5.5 defaults concise. Both can be tuned, but the starting point differs.
- **Reasoning controls** — Opus 4.7 removed manual thinking budgets in favor of adaptive only; GPT-5.5 still allows manual `reasoning_effort` levels
- **Pushback disposition** — Opus volunteers "the requested approach has a problem" more readily than GPT does. GPT will do it on instruction; Opus does it natively.

## Practical implications

For AGENTS.md / CLAUDE.md authors:

- Audit existing files against the three layers. Anything model-intrinsic should be tagged with the model it applies to. Anything that's craft for an older model should be removed or replaced.
- Resist the urge to make instruction files comprehensive. Each line costs context budget; lines that encode a closed gap are pure cost.
- When a new model lands, the right move is to *strip back* the file rather than add more. If the model is more capable, less coaching is needed.
- Keep model-version-tagged profiles for older models that genuinely need heavier scaffolding (e.g., GPT-5.1 with stricter "consider alternatives" coaching that 5.5 doesn't need).

For prompt engineers reading vendor guidance:

- Prefer the official model-specific guide (e.g., Codex prompting guide for Codex CLI work) over generic "best practices" articles
- Cross-reference vendor advice against the three layers — if both vendors say it, it's probably physics; if only one does, check whether it's a model-intrinsic feature
- When advice from different sources contradicts, the contradiction usually traces to a model-intrinsic difference, not bad advice on either side

For teams porting prompts between models:

- Old prompts encode the old model's gap. Don't trust them on the new model.
- Test the new model with no scaffolding first, see what it does well natively, then add only what's still needed
- Keep the old prompts as profiles for environments where the old model is still in use, but don't apply them to new model deployments
