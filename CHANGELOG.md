# Changelog

## 1.0.0 — 2026-08-31 (public launch)

First public release of MemStore: durable, private memory for AI agents —
fully offline, no models, no embeddings, no cloud.

- **Durable memory engine** — LMDB storage, Tantivy/BM25 retrieval with
  heuristic rerank and a raw-text fallback. Temporal truth (`valid_from` /
  `valid_until`), write-time contradiction detection, git-style branching
  with 3-way merge, trust scoring, budget-aware evidence compression.
- **14 MCP tools** — store, update, delete, search, drill-down, temporal
  query, contradictions, versioning, tasks, feedback, graph exploration,
  audit, configuration, admin. Tenants follow projects, so every MCP client
  shares one memory root.
- **Conflict-safe updater** — `install.py --update <zip>` diffs a shipped
  `MANIFEST.sha256` against the installed tree: unmodified files update in
  place, files you modified are preserved (the new version lands next to
  yours as `*.new-<version>` with a merge report), deleted files are
  restored, and every replaced original is backed up first.
- **Licensing** — one-time Lemon Squeezy activation at install, monthly
  silent re-validation, offline fallback; updates never burn activation
  slots.
- **Memory River** — a local dashboard to browse what your agents remember.
- **Benchmarks** — comparison harnesses are in the pipeline for the public
  repo; run the product's own `doctor` and smoke tests on your machine
  before committing.

### Notes

- Internal development history (pre-1.0.0) is consolidated here; the
  engine, storage format and tool surface are stable as of this release.
- Updates are never automatic: `python -m memstore update-check` and the
  monthly hint in `doctor` only flag new versions.
