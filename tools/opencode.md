# OpenCode — verified facts (checked 2026-10-05, OpenCode v1.18.34)

Read from the sources on the date above: the published docs (opencode.ai/docs — config, plugins, SDK, CLI, network), the repository (`github.com/anomalyco/opencode`, branch `dev`; `sst/opencode` redirects there), the plugin package's type definitions, and models.dev. Latest release v1.18.34 (Sep 30 2026), bundling `@ai-sdk/azure` 3.0.93 and `@ai-sdk/openai` 3.0.88.

## Models on Azure OpenAI / Foundry

- models.dev lists the GPT-5.6 Sol/Terra/Luna, GPT-6 Astra/Sol/Luna and GPT-6.1 Sol families for both the `openai` and `azure` providers (1,050,000 context / 922,000 input / 128,000 output). OpenCode turns each model's `reasoning_options` into selectable variants (`low` … `xhigh`, `max`; the Luna tier adds `none`) and lowers per-model `options.reasoningEffort`, `reasoningSummary`, `textVerbosity` to the Responses API's `reasoning.effort`, `reasoning.summary`, `text.verbosity`.
- Model id form: `azure/<catalog id>` (e.g. `azure/gpt-6.1-sol`). The Azure **deployment name must equal the catalog id** unless you map it: `"provider": { "azure": { "models": { "gpt-6.1-sol": { "id": "<your-deployment-name>" } } } }`.
- Provider: id `azure`, `npm: "@ai-sdk/azure"`; resource from `options.resourceName` (or `options.baseURL`); API key via `options.apiKey = "{env:AZURE_OPENAI_API_KEY}"`; Entra ID via `az login` is supported. `apiVersion` is not an OpenCode option (unknown options pass through to the SDK). Whether a given deployment accepts `xhigh`/`max` is decided server-side by Azure.
- A sensible pairing: a Sol-tier model as `model`, a Luna-tier one as `small_model` (titles, summaries) and for cheap read-only agents.

## `opencode.json` schema changes

- Moved **out** of `opencode.json`: `tui`, `keybinds`, `theme` → `tui.json` (`OPENCODE_TUI_CONFIG`); auto-migrated with a warning.
- Deprecated aliases (still accepted): `mode` → `agent`, `autoshare` → `share`, `reference` → `references`, agent `tools` → `permission`, agent `maxSteps` → `steps`.
- Current keys: `agent`, `attachment`, `autoupdate` (`true|false|"notify"`), `command`, `compaction` (`auto`, `prune`, `tail_turns`, `preserve_recent_tokens`, `reserved`), `default_agent`, `disabled_providers` (beats `enabled_providers`), `formatter`, `instructions[]`, `lsp`, `mcp`, `model`, `permission`, `plugin[]`, `provider`, `references`, `server`, `share` (`manual|auto|disabled`), `shell`, `skills` (`paths`, `urls`), `small_model`, `snapshot`, `subagent_depth`, `tool_output` (`max_lines` 2000, `max_bytes` 51200), `tools`, `username`, `watcher`.
- Agent entry: `model`, `variant`, `temperature`, `top_p`, `prompt`, `description`, `mode` (`primary|subagent|all`), `hidden`, `color`, `steps`, `options`, `permission`, `disable`; unknown keys pass through as model options.
- Permissions: `ask|allow|deny`; `read/edit/glob/grep/list/bash/task/external_directory/lsp/skill` accept pattern objects (`"git push *": "deny"`, last match wins); `webfetch/websearch/todowrite/question/doom_loop` are scalar; defaults allow except `doom_loop` and `external_directory` = ask and `.env` reads denied; `--auto` flag and `OPENCODE_PERMISSION` (inline JSON) override.
- Compaction: the default "preserve recent" budget is 25 % of the usable window clamped to 2,000–15,000 tokens.
- A **V2 config shape** (`permissions`, `plugins`, `providers`, `agents`, `compaction.keep.tokens`, `compaction.buffer`, `model.variant`) exists in the source and the current CLI errors on V2 `permissions` ("Use V1 "permission" rules or run opencode2"). Stay on the V1 keys.

## Instructions, skills, context

- Project instructions: `AGENTS.md`, else `CLAUDE.md` (first match walking up; `CONTEXT.md` deprecated). Global: `~/.config/opencode/AGENTS.md`, else `~/.claude/CLAUDE.md`. `instructions[]` adds globs or URLs (5 s fetch timeout).
- Instructions are injected into the **system prompt on every LLM step** (`system = [env, instructions, mcp, skills]`), so they survive compaction. The "instruction loss at compaction" issue of early 2026 is historical. No size cap was found.
- Skills discovery: `.opencode/skills/` (or `skill/`), `~/.config/opencode/skills/`, `.claude/skills/`, `~/.claude/skills/`, `.agents/skills/`, `~/.agents/skills/`, plus `skills.paths` / `skills.urls`. `OPENCODE_DISABLE_EXTERNAL_SKILLS` drops the `.claude` and `.agents` locations; `OPENCODE_DISABLE_CLAUDE_CODE`, `…_PROMPT`, `…_SKILLS` are finer switches. One shared `skills/` folder therefore serves OpenCode, Codex (`[[skills.config]]`) and Claude Code.

## Plugins are the hook mechanism

Local JS/TS plugins in `.opencode/plugins/` or `~/.config/opencode/plugins/` auto-load (dependencies via `.opencode/package.json` + `bun install`); npm plugins go in `plugin: []`. A plugin exports an async function `({ project, client, $, directory, worktree }) => ({ hooks })`.

- Hooks: `event` (receives `session.idle`, `session.created`, `session.updated`, `session.compacted`, `session.error`, `permission.asked/replied`, `message.updated`, TUI events …), `config`, `tool`, `auth`, `provider`, `chat.message`, `chat.params`, `chat.headers`, `permission.ask` (`output.status = ask|deny|allow`), `command.execute.before`, `tool.execute.before/after`, `shell.env`, `tool.definition`, `experimental.chat.messages.transform`, `experimental.chat.system.transform`, `experimental.provider.small_model`, `experimental.session.compacting` (push strings onto `output.context`, or set `output.prompt`), `experimental.compaction.autocontinue`, `experimental.text.complete`, `dispose`.
- The `client` is the SDK: `client.session.prompt({ path: { id }, body: { parts: [{ type: "text", text }], noReply?: true } })` — with `noReply: true` the text is injected as context without a model reply ("useful for plugins"); `client.session.messages({ path })` reads the transcript; `client.tui.showToast`, `client.app.log`.
- **A Stop-hook equivalent:** `event` on `session.idle` → check the notes file's age → `client.session.prompt` with a nudge, guarded by a per-session cooldown so it cannot loop; `session.created` → inject the notes file with `noReply`; `experimental.session.compacting` → push the rule into the compaction context. A working version (`opencode/plugins/session-notes.js`, exercised against a fake client) is in [jblenman/ai-agent-templates](https://github.com/jblenman/ai-agent-templates).

## Air-gapped / restricted use

`share: "disabled"`, `autoupdate: false`; environment: `OPENCODE_DISABLE_AUTOUPDATE`, `OPENCODE_DISABLE_MODELS_FETCH` (+ `OPENCODE_MODELS_URL` / `OPENCODE_MODELS_PATH`), `OPENCODE_DISABLE_LSP_DOWNLOAD`, `OPENCODE_DISABLE_DEFAULT_PLUGINS`, `OPENCODE_DISABLE_EXTERNAL_SKILLS`, `OPENCODE_PURE` / `--pure`, `OPENCODE_DISABLE_PROJECT_CONFIG`, `OPENCODE_CONFIG` / `_DIR` / `_CONTENT`, `OPENCODE_AUTO_SHARE`; proxies through `HTTPS_PROXY` / `NO_PROXY`, custom CAs through `NODE_EXTRA_CA_CERTS`. Managed config directories: `%ProgramData%\opencode`, `/etc/opencode`, `/Library/Application Support/opencode` (and macOS MDM `ai.opencode.managed`).

## Sources

- https://opencode.ai/docs/config · …/docs/plugins · …/docs/sdk · …/docs/cli · …/docs/network · https://opencode.ai/config.json
- https://github.com/anomalyco/opencode (`packages/core/src/v1/config/config.ts`, `provider-options.ts`, `packages/plugin/src/index.ts`, `packages/opencode/src/session/prompt.ts`, `compaction.ts`, `provider/transform.ts`, `effect/runtime-flags.ts`, `config/v2-compat.ts`; releases)
- https://models.dev
