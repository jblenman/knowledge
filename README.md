# knowledge

A small public knowledge garden — notes I keep on languages, tools, and patterns. Topic-based, not chronological. Each file is meant to stand on its own.

This is the public-facing subset of a larger personal KB. The published files have no personal, machine-specific, or work-context content. New entries will land here as topics come up.

## Browse

Start at **[index.md](index.md)** for a topic-grouped table of contents.

## Query it as a RAG corpus

Designed to pair with [local-rag](https://github.com/jblenman/local-rag), my hybrid (vector + BM25) RAG library. Clone both:

```bash
git clone https://github.com/jblenman/local-rag.git
git clone https://github.com/jblenman/knowledge.git

cd local-rag
pip install -r requirements.txt
ollama pull nomic-embed-text

# Point the indexer at this corpus instead of the demo
RAG_CORPUS_ROOT=../knowledge python kb_index.py
RAG_CORPUS_ROOT=../knowledge python kb_search.py "what is reciprocal rank fusion"
```

The two repos are independent — you can use this corpus with any RAG/search tool, and `local-rag` can index any markdown corpus.

## Contents at a glance

- **`languages/`** — Per-language notes: gotchas, idioms, version compatibility. Two stubs (`javascript.md`, `swift-ios.md`) are placeholders waiting for content.
- **`patterns/`** — Reusable design patterns and reference material: SQL Server encryption, view semantics, data-security architecture, LLM prompting philosophy, Windows admin idioms.
- **`tools/`** — Configurations, templates, and reference material for specific tools. Currently just an AGENTS.md template for Codex CLI; more to come.

## Contributing back to your own version

Fork it, point your own `local-rag` instance at the fork, add files in the same structure. No build step — just markdown.

## License

MIT — see [LICENSE](LICENSE).
