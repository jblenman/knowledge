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

## Tools

| File | Topics |
|------|--------|
| [tools/codex-agents-template.md](tools/codex-agents-template.md) | Reusable AGENTS.md template for Codex CLI — reasoning, workflow, communication, git safety, code quality |
| [tools/goose.md](tools/goose.md) | Goose AI agent config — locations, format, env var precedence, remote Ollama setup, tool shim for weak tool-calling models |

## See also

- [jblenman/local-rag](https://github.com/jblenman/local-rag) — the local hybrid RAG library this corpus is designed to be queried with
- [jblenman/hivemind](https://github.com/jblenman/hivemind) — local Ollama fleet router and agentic coding assistant
