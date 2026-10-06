# Codex CLI — verified facts (checked 2026-10-05, Codex CLI 0.160.1)

Everything below was read from the official sources on the date above: the config reference and config-advanced pages, `config-schema.json`, the hooks page, the AGENTS.md page, the Codex changelog, the OpenAI API changelog and model pages, and the Azure AI Foundry model page. Codex changes monthly; re-check before relying on a detail.

## Models: family + tier

OpenAI's 2026 naming is a *family* (GPT-5.6, Jul 2026; GPT-6, Sep 2026; GPT-6.1, Sep 2026) and a *tier*:

| Tier | Meaning | Examples |
|---|---|---|
| **Sol** | most capable | `gpt-5.6-sol`, `gpt-6-sol`, `gpt-6.1-sol` (Codex's default since 0.159) |
| **Terra** | balanced | `gpt-5.6-terra` |
| **Luna** | efficient, high-volume — OpenAI describes it as roughly the nano tier; in Codex it replaced `gpt-5.4-mini` | `gpt-5.6-luna`, `gpt-6-luna` |

`gpt-5.6-luna`: 1.05M context, 128K output, knowledge cutoff Feb 2026, reasoning efforts `none`/`low`/`medium` (default)/`high`/`xhigh`/`max`; on Azure as `gpt-5.6-luna (2026-07-09)`, Global Standard in most regions, Responses or Chat Completions (tools with reasoning need Responses). GPT-5.5 retires from Codex on Oct 14 2026.

**Practical consequence:** "max effort" on a Luna-tier model is still a nano-class model. If a Luna deployment is all you have, expect a small model's habits (first plausible answer, thin investigation, early hand-backs) and give it a higher reasoning effort plus explicit evidence rules (below). If it keeps handing back, the fix is a Sol-tier deployment, not more prompt text.

## `config.toml` keys that changed (what breaks an older config)

| Older key / value | Now |
|---|---|
| `[profiles.<name>]` tables, `profile = "<name>"` | **Retired in 0.134.** A profile is a separate file `~/.codex/<name>.config.toml`, loaded as an overlay above `config.toml` by `codex --profile <name>`; it holds only the keys that differ. A `[profiles.*]` table is silently ignored. |
| `approval_policy = "untrusted"` | **Unsupported.** Values: `on-request` (interactive), `never` (non-interactive), or `{ granular = { sandbox_approval, rules, mcp_elicitations, request_permissions, skill_approval } }`. `on-failure` is deprecated. `approvals_reviewer = "auto_review"` routes eligible prompts to a reviewer subagent. |
| `features.child_agents_md` | Never existed. Sub-agents inherit model, effort, sandbox, MCP servers and skills from the parent; whether they receive the AGENTS.md chain is not documented. |
| `features.js_repl` | A retired switch (rejected under Work Cloud). |
| `features.web_search`, `web_search_cached`, `web_search_request` | Deprecated legacy toggles; use the top-level `web_search = disabled \| cached \| indexed \| live`. |
| `personality = "friendly" \| "pragmatic"` | The schema marks the styles deprecated ("no longer select a style"); the reference still lists the key. Put tone in AGENTS.md instead. |
| `model_max_output_tokens` | Absent from the reference, sample and schema. |
| `shell_environment_policy.exclude` / `include_only` | Legacy; use `shell_environment_policy.filters`. |
| `agents.max_threads` | `agents.max_concurrent_threads_per_session`. |
| `features.codex_hooks` | `features.hooks`. |

Still valid and useful: `model_reasoning_effort` (a free string whose allowed values depend on the model: `low`…`xhigh`, `max`; `ultra` only on GPT-6 Astra), `model_reasoning_summary` (`auto|concise|detailed|none`), `model_verbosity` (`low|medium|high`), `plan_mode_reasoning_effort`, `review_model` (defaults to the session model), `tool_output_token_limit`, `project_doc_max_bytes` (default 32768), `project_doc_fallback_filenames` (e.g. `[".claude/CLAUDE.md", "CLAUDE.md"]`), `model_auto_compact_token_limit` (+ `_scope`), `model_context_window`, `[history] persistence = "save-all"`, `[memories]`, `[features] undo / multi_agent / memories / prevent_idle_sleep / request_permissions / codex_git_commit / hooks`, `[tui] status_line = ["model-with-reasoning", "context-remaining", "git-branch"]`.

**TOML ordering:** every root-level key must appear before the first `[table]` header; a key placed after one belongs to that table and is silently ignored. The most common Codex config bug.

### Azure OpenAI provider block (verbatim shape from config-advanced)

```toml
[model_providers.azure]
name = "Azure"
base_url = "https://YOUR_RESOURCE.openai.azure.com/openai"
env_key = "AZURE_OPENAI_API_KEY"
query_params = { api-version = "2025-04-01-preview" }
wire_api = "responses"          # the only supported value
request_max_retries = 4
stream_max_retries = 10
stream_idle_timeout_ms = 300000
```

`model` is then the Azure **deployment name**. Azure's v1 GA API path (`/openai/v1`) needs no `api-version`.

## AGENTS.md: how it loads, what OpenAI now recommends

- Global: `~/.codex/AGENTS.override.md`, else `~/.codex/AGENTS.md` (first non-empty). Project: walks from the Git root down to the working directory, one file per directory (override, then `AGENTS.md`, then the fallback names), concatenated root → cwd, up to `project_doc_max_bytes` (32 KiB by default). Rebuilt on every run, no cache.
- OpenAI's guidance for GPT-5.6 and GPT-6 (their "latest model" guide and the GPT-6 Astra blog, Sep 2026): leaner prompts score higher (+10–15 % on their evals) and spend far fewer tokens (−41–66 %); state autonomy boundaries **once** (repeated "ask first" language causes unnecessary approval requests); the models are more concise by default; do not ask them to "think harder". The newest models are **more** sensitive to instructions in AGENTS.md and skills, and conflicting instructions make them pause early — audit for contradictions.
- Their persistence pattern, in plain words: bias toward action; persist until the user's intended goal is complete; a partial or "helpful enough" result is not done; come back with a concrete, reviewable result; grant permission for safe workflows up front so the model does not stop to ask.

### The rule the official guidance does not contain: evidence for tool results

Observed with a Luna-tier model on an Azure CLI task: `az … --query "<malformed>"` returned `[]`, and the agent reported "your account has no access" — the unfiltered command listed everything. Nothing in OpenAI's prompting docs covers it, so the AGENTS.md needs it explicitly:

- **Empty is not "no access".** Likelier causes, in order: the filter or query; the scope (subscription, tenant, resource group, branch, directory); a quiet failure (exit code, stderr, a warning, a truncated page); only then permission — and permission is named only next to an explicit authorization error, quoted.
- **Bisect before concluding.** Rerun a filtered, queried or piped command in its simplest form, then add one piece back at a time. Test a JMESPath, jq, regex or WHERE clause on a row you have already seen before trusting its empty result.
- **Prove the claim.** Any statement about the environment ("no access", "not installed", "does not exist", "already configured") carries the exact command and the output line that shows it.
- **Three different attempts before a hand-back.** The same command again is a retry; removing a variable (filter, scope, flag, extension, syntax) is an investigation.
- **Separate verified from assumed** in every finding.

A complete AGENTS.md with these sections, plus the `azure-cli` skill that encodes the Azure specifics (JMESPath strings must be single-quoted; double quotes inside a filter return empty output — Microsoft's own words; `datafactory` and `resource-graph` are extensions, `synapse` is core), is in [jblenman/ai-agent-templates](https://github.com/jblenman/ai-agent-templates).

## Hooks (stable; the same contract as Claude Code's)

`features.hooks = true`, then `~/.codex/hooks.json` (or inline `[hooks]` in `config.toml`; one representation per layer; a repo's `.codex/hooks.json` loads only in trusted projects).

- Events: `SessionStart`, `SessionEnd`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PreCompact`, `PostCompact`, `SubagentStart`, `SubagentStop`, `Stop`, `Interrupt`.
- Input: one JSON object on stdin — `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `model`, plus `turn_id`, `stop_hook_active`, `trigger` (manual/auto), tool fields.
- Output: JSON on stdout. `Stop` → `{"decision": "block", "reason": "..."}` makes Codex continue the turn with the reason as a new prompt (`stop_hook_active` is true on the second pass, so a hook never loops); exit 2 + stderr does the same. `PreCompact` → `{"continue": false}` stops a compaction. `SessionStart` → `{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "..."}}` (capped by `additionalContextLimit`). `PreToolUse` → `permissionDecision` allow/deny with `updatedInput` to rewrite a command.
- `commandWindows` (JSON) / `command_windows` (TOML) gives a Windows-only command. Do not rely on shell variable expansion in it — a `%USERPROFILE%` form failed on the first prompt in testing; `py -c "import os,runpy; runpy.run_path(os.path.expanduser('~/.codex/hooks/session_notes.py'), run_name='__main__')" stop` lets Python resolve `~` and works whichever shell Codex uses. A changed hook definition must be re-trusted in `/hooks`. Default timeout 600 s (`SessionEnd`/`Interrupt` 1–3 s). Non-managed hooks must be reviewed and trusted once with `/hooks` (hash-pinned; a changed hook is skipped until re-trusted); `--dangerously-bypass-hook-trust` for vetted automation.
- Codex's own docs example is literally "Loading session notes" at `SessionStart` — a session-record that survives compaction and restarts is the intended use. A ready-made one (`SessionStart` hands the notes file to the model, `Stop` continues the turn once while it is stale or missing, `PreCompact` records the compaction) is `codex/hooks/session_notes.py` in the templates repository.

## Windows: `CreateProcessAsUserW failed: N`

Codex's native Windows sandbox runs every shell command as a restricted local user (`CodexSandboxOnline` / `CodexSandboxOffline`). When that user cannot start the process, the command never runs (`Failed to create unified exec process: CreateProcessAsUserW failed: N`) and the model falls back to its own `read_file` / `apply_patch` tools, so reads still get done — or it switches to `cmd.exe /c` for everything, which is the tell.

**The usual cause — `openai/codex` issue #35871, 16 independent confirmations: the Microsoft Store (MSIX) build of PowerShell 7.** Codex resolves `pwsh` through the per-user App Execution Alias (`%LOCALAPPDATA%\Microsoft\WindowsApps\pwsh.exe`), and Windows refuses to launch a packaged binary under the sandbox's restricted token. A `winget install PowerShell` that resolved to the `msstore` source produces exactly this layout; the error line's `cmd="C:\Program Files\WindowsApps\Microsoft.PowerShell_…\pwsh.exe"` segment is the confirmation. Every PowerShell-form command (`Get-Content -Raw`, `rg | Select-Object`, `Get-Date`) fails with error `5`; the `unelevated` backend prints it as `-1073283067`, which is `0xC0070005` — a wrapped access-denied, not a real Win32 code (issue #35958). `cmd.exe` is a plain `System32` binary and launches normally.

Fixes, in order of least privilege:
1. Turn off the `pwsh.exe` app execution alias (Settings → Apps → Advanced app settings → App execution aliases), or remove `WindowsApps` from the PATH Codex starts with. Codex then falls back to Windows PowerShell 5.1 (`C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`), which the sandbox launches (20/20 in the thread). No admin rights needed. Verified end to end on a corporate laptop with the Store build 7.6.6: after the alias was disabled the sandbox ran PowerShell commands again and the model stopped routing everything through `cmd.exe`.
2. Install PowerShell 7 from the MSI (`C:\Program Files\PowerShell\7\pwsh.exe`) ahead of the Store build in PATH — one reporter saw Codex still pick the alias, so verify.
3. `windows.sandbox = "elevated"` after `codex sandbox setup --elevated`: Codex's own fallback to an unpackaged PowerShell exists in the source but only for that backend. Note the costs of `unelevated` measured in the thread: PowerShell drops to ConstrainedLanguage mode and the read-only sandbox tier is unavailable.

Other codes: `2` = the program cannot be found or executed by the sandbox user, `1312` = no logon session. Verify with `codex doctor` and `codex sandbox windows -- powershell -NoProfile -Command "Get-Date"`. Last resort: `sandbox_mode = "danger-full-access"` while keeping `approval_policy = "on-request"`. (`features.unified_exec` is "enabled by default except on Windows" per the reference.)

## Skills

A skill is a folder with a `SKILL.md` (frontmatter `name` + `description`, then the procedure). Codex picks one when a task matches its description, or you invoke it with `$name`. `~/.codex/skills/<name>/SKILL.md` is picked up without any config entry (observed on 0.160; the reference documents only the `[[skills.config]]` form). `[[skills.config]] path = "<folder>"` / `enabled = true` in `config.toml` registers a folder from anywhere on disk, so one shared `skills/` folder serves Codex, OpenCode and Claude Code. `skills.max_context_tokens` caps the catalog shown to the model (2 % of the context window by default, at most 10,000).

## Sources

- https://developers.openai.com/codex/config-reference · …/codex/config-advanced · …/codex/config-schema.json · …/codex/hooks · …/codex/agent-configuration/agents-md · …/codex/agent-configuration/subagents · …/codex/changelog
- https://developers.openai.com/api/docs/changelog · …/api/docs/models/gpt-5.6-luna · …/api/docs/guides/latest-model · the "Rethinking skills and prompts for GPT-6 Astra" blog post
- https://learn.microsoft.com/en-us/azure/ai-foundry/openai/concepts/models · …/cli/azure/query-azure-cli · …/cli/azure/azure-cli-extensions-list
