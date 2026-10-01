# Knowledge Index

A small, growing collection of technical reference notes. Topic-based, not chronological. Each file is meant to be useful on its own.

This corpus is indexable and queryable with [local-rag](https://github.com/jblenman/local-rag) — `RAG_CORPUS_ROOT=./ python /path/to/kb_index.py` gives you hybrid vector + BM25 search across everything below.

## Languages

| File | Topics |
|------|--------|
| [languages/python.md](languages/python.md) | Python 3.8 vs 3.9+ compatibility, Windows CP1252 gotchas, subprocess encoding |
| [languages/python-from-csharp.md](languages/python-from-csharp.md) | Python idioms with C# / .NET analogues — translation guide for modules, language constructs, collections, paths, time |
| [languages/csharp-dotnet.md](languages/csharp-dotnet.md) | Async/parallel — sequential await vs `Task.WhenAll` vs true parallelism; cursor-based paging helper; `HttpClient` with conditional proxy |
| [languages/javascript.md](languages/javascript.md) | *(placeholder — to grow as patterns come up)* |
| [languages/swift-ios.md](languages/swift-ios.md) | *(placeholder — to grow as patterns come up)* |

## Patterns

| File | Topics |
|------|--------|
| [patterns/llm-prompting-philosophy.md](patterns/llm-prompting-philosophy.md) | Three-layer model (model-intrinsic / general LLM physics / craft) for evaluating prompt advice; how the line shifts as models improve |
| [patterns/data-security-fundamentals.md](patterns/data-security-fundamentals.md) | Encoding vs encryption, defense-in-depth threat modeling, where keys live, authenticated encryption, compliance drivers |
| [patterns/sql-server-encryption.md](patterns/sql-server-encryption.md) | Column-level (symmetric + cert), Always Encrypted, application-layer, TDE — what each protects against; `EncryptByKey` output structure; authenticator parameter; ciphertext detection |
| [patterns/sql-server-views.md](patterns/sql-server-views.md) | T-SQL view semantics, predicate pushdown, what `WITH SCHEMABINDING` actually does (and doesn't), indexed views, lookup-table pattern for UI dropdowns |
| [patterns/windows-admin.md](patterns/windows-admin.md) | Detached background processes, killing elevated processes, PowerShell auto-transcripts, Google Drive `.symlink` false positives, Git Bash on Windows quirks |
| [patterns/chrome-extension-spa-leaks.md](patterns/chrome-extension-spa-leaks.md) | Dark Reader memory leak on Material/Angular SPAs: heap-snapshot signature, two-path source-code root cause (`StyleManager.destroy` cleanup gap + `imageSelectorQueue` async Promise retention), sanitization before sharing, filed upstream as `darkreader/darkreader#14164` |
| [patterns/heap-snapshot-streaming-analysis.md](patterns/heap-snapshot-streaming-analysis.md) | Streaming Chrome heap-snapshot analysis for multi-GB captures that DevTools can't reload: `ijson` + pre-allocated `array.array`, two-pass strings + nodes, ~300 MB peak memory, sanitization technique; public gist with the scripts |
| [patterns/oss-maintainer-bug-reports.md](patterns/oss-maintainer-bug-reports.md) | Writing upstream bug reports that land: addressing prior pushback, crediting other reporters, evidence-before-conclusion framing, leading suggested fixes with the codebase's existing patterns, conditional PR offers, honest AI-assistance acknowledgment, not pre-judging maintainer's costs |
| [patterns/3d-printing.md](patterns/3d-printing.md) | STL diagnostic decision tree, dual-silhouette / shadow-art models (don't "repair"), trimesh mesh-stats snippet, how STL repair tools actually work, splint-repair strategy for MDF parts, captive-grommet sandwich pattern, structural PETG print settings, reusable OpenSCAD modules |

## Tools

| File | Topics |
|------|--------|
| [tools/codex-agents-template.md](tools/codex-agents-template.md) | Reusable AGENTS.md template for Codex CLI — reasoning, workflow, communication, git safety, code quality |
| [tools/goose.md](tools/goose.md) | Goose AI agent config — locations, format, env var precedence, remote Ollama setup, tool shim for weak tool-calling models |
| [tools/claude-code-action-guard.md](tools/claude-code-action-guard.md) | Template: a Claude Code `PreToolUse` hook that refuses or asks about actions against one web application — hook contract (stdin JSON, deny/ask JSON out, fail open), a shell reader for curl/wget/httpie/PowerShell request methods, path rules, Claude in Chrome tab memory, MCP tool rules; full source, installer and 22 tests in code blocks with setup and adaptation notes |
| [tools/claude-code-knowledge-base-local.md](tools/claude-code-knowledge-base-local.md) | Template: a knowledge base for Claude Code on one machine, without a git repository — a Claude Code session sets it up from the document itself (four questions with options: this project or every project, the application's name, keep your own QA skill, keep the files out of git); a two-line rule file loads an entry file and an index; UI workflow cards for ad-hoc QA with an optional one-call script; `qa` and `kb-capture` skills; a PowerShell validator; what makes a test run quick, with measurements; how to move it to every project or into a shared repository; all 19 files in code blocks with a setup script |
| [tools/claude-code-team-knowledge-base.md](tools/claude-code-team-knowledge-base.md) | Template: a team knowledge base that Claude Code sessions load when they start — a git repository every developer clones; a two-line rule file pulls in an entry file and an index; UI workflow cards (steps, locators, an expected result per step, optional one-call script) for ad-hoc QA; `qa` and `kb-capture` skills; a PowerShell validator for pull requests; Azure DevOps pipeline and pull request template; how a local copy becomes the repository; all 22 files in code blocks with an unpack snippet, setup and install steps, and a measured comparison with and without the cards |

## See also

- [jblenman/local-rag](https://github.com/jblenman/local-rag) — the local hybrid RAG library this corpus is designed to be queried with
- [jblenman/hivemind](https://github.com/jblenman/hivemind) — local Ollama fleet router and agentic coding assistant
