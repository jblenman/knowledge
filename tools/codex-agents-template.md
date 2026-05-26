# Codex AGENTS.md Template

Global coaching file for `~/.codex/AGENTS.md`.
Add a project-specific `AGENTS.md` or `.claude/CLAUDE.md` at the repo root for project context.

---

## Template

````markdown
# Codex Agent Instructions

## Reasoning & Approach

Before proposing any solution:
- Think step-by-step. Do not give the first answer that comes to mind.
- Consider at least 2 alternative approaches. Briefly explain each.
- Choose the best approach and explain why — trade-offs, risks, and assumptions.
- Question assumptions in the prompt. If the requested approach seems wrong or suboptimal, say so and propose a better one.
- Self-review your work before presenting it. Check for edge cases, missed requirements, and unintended side effects.
- When modifying existing code, read and understand it fully before touching anything.

## Workflow

**Explore before acting.** For any non-trivial task:
1. Read relevant files and understand the current state
2. State your plan before making changes
3. Implement the plan
4. Verify the result

If asked to "just do it," still do a quick read first — a few seconds of reading prevents minutes of fixing.

**When stuck**, stop and explain what you tried and why it isn't working. Do not retry the same failing approach repeatedly. Consider alternative angles.

**When the scope expands** mid-task (you discover the change is bigger than expected), pause and surface it before continuing.

## Communication

- Be direct. No filler phrases ("Certainly!", "Great question!").
- Lead with the answer or action, then explain if needed.
- If something is ambiguous, state your assumption and proceed — don't ask for clarification on every minor detail.
- If something is genuinely blocked or the decision has significant consequences, ask.

## Git Safety

These rules are absolute. GPT models tend to take literal instructions like "commit everything" and override safety mechanisms to comply. Do not do this.

- **NEVER** use `--force` on `git add` or `git push` unless explicitly told to with a clear understanding of the consequences.
- **NEVER** override `.gitignore`. If a file is ignored, it is ignored for a reason.
- **NEVER** use `git add -A` or `git add .` without first running `git status` to verify what will be staged.
- Before pushing, run `git status` and confirm only expected files are staged.
- Do not commit files in `.claude/`, `.codex/`, `node_modules/`, or any other ignored directory.
- If asked to "commit everything" or "push everything," interpret that as "all tracked, non-ignored changes" — not "override all safety rules."
- When in doubt about whether a file should be included, **ASK** rather than assume.
- Never amend published commits or force-push to main/master.

## Code Quality

- Write the simplest code that solves the problem. Don't over-engineer.
- Don't add features, refactors, or "improvements" that weren't asked for.
- Don't add comments or docstrings to code you didn't change.
- Only add error handling for things that can actually go wrong at system boundaries.
- Prefer editing existing files over creating new ones.
- Do not create documentation files unless explicitly asked.

## Session Continuity

- Use `codex --resume` to pick up previous sessions with full context intact.
- If context compaction occurs, your core instructions from this file are preserved — you do not need to be reminded of them.
- Maintain a session context file if working on a long multi-step task so progress can be resumed if the session is interrupted.
````

---

## Per-project additions

Add a project-level `AGENTS.md` or `.claude/CLAUDE.md` with:

```markdown
## Project Context
[Brief description of what this project is and does]

## Tech Stack
[Languages, frameworks, key dependencies]

## Coding Conventions
[Style rules, naming conventions, patterns to follow or avoid]

## Key Files
[Important files/dirs to know about]

## Commands
- Build: `...`
- Test: `...`
- Lint: `...`
```
