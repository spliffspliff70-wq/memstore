# MemStore_v2 Architecture — How Memory Is Stored and Found

MemStore_v2 is a **fully model-free** system: everything below runs offline
with zero neural models, zero downloads, sub-second startup.

## The LMDB database map (per tenant, per branch)

One file per branch: `data/tenants/tenant_<id>.db` (main) and
`tenant_<id>_<branch>.db` (other branches). Each file is ONE LMDB
environment with named sub-databases:

| Sub-DB | Key → Value | Purpose |
|---|---|---|
| `memories` | `memory_id (u64)` → msgpacked `MemoryRecord` | The source of truth — every memory. |
| `embeddings` | `memory_id` → fp32 bytes | Kept only for v1 import compat; unused by v2 retrieval. |
| `graph_edges` | `memory_id` → `GraphEdge[]` (dupsort) | Typed edges: `related_to`, `contradicts`. |
| `adjacency` | `memory_id` → packed neighbor ids | Fast 1-hop expansion in retrieval. |
| `tag_index` | `tag` → memory_id (dupsort) | Tag → records; also powers auto-linking & lookups. |
| `indices` | `tt_<ms>`+id → memory_id | Creation-time ordered index (temporal windows, recent). |
| `state` | key → scalar | `next_memory_id`, `schema_version`, `memory_count`. |

A full-text Tantivy index lives next to it per branch:
`tenant_<id>_index/` with a MemStore-owned `terms` field (normalized tokens
from `index/keyword_extractor.normalize_terms`, identical pipeline for
documents and queries) plus stored `content`, `tags`, `memory_type`,
`provenance`, `created_at`.

## One memory record (what gets stored)

```text
MemoryRecord
  memory_id u64          tenant_id str        version_id int      protected bool
  content str            memory_type enum     tags set[str]
  provenance enum        confidence 0..1      trust_score 0..1    specificity 0..1
  created_at epoch       valid_from epoch     valid_until epoch?  before/after_value
  accessibility 0..1     last_accessed epoch  access_count int
  related_memory_ids     contradicts_ids      source_memory_ids
```

`memory_type` decides its lifecycle half-life (episodic=1d, procedural=7d,
semantic=30d, document=90d, constraint=never decays, …). All date/time fields
are **assigned automatically at store time**: `created_at` is now;
`valid_from`/`valid_until` are extracted from the content by the temporal
normalizer when it mentions dates ("on 2026-03-15", "yesterday"), else
default to `created_at` — that's what makes ordinal queries
("first/last/earliest") work over *event* time, not just insertion order.

## End-to-end flow

```text
INGEST (path/text/transcript/watch)
  → SemanticChunker (heading-aware split, overlap, tail-merge)
  → IngestionProcessor → MemoryRecord(type=DOCUMENT, tags=source/heading)
  → (optional MemoryEnricher/FactExtractor for agent/agent-less flows)
  → MemStoreDB.store_memory()
      · allocates id atomically      · auto-links via shared tags
      · updates tag/temporal/graph indices + adjacency
  → Tantivy add_document (terms field, MemStore-normalized)
  → AuditLogger + vault mirror (opt-in) + metrics

STORE (tool)
  → provenance inference from source   → tags (given) 
  → temporal normalize (dates → valid_from/until)
  → TrustScorer.score (persisted)      → trust-gate check (opt-in)
  → same write path as above

SEARCH (tool)
  → mode classify (exact/hybrid)
  → Tantivy/BM25 candidates (normalized terms, x5k_>50 docs)
     (<20 hits → graph 1-hop expand via adjacency)
  → heuristic rerank: BM25 base ×(1+kw+sparse learned)
     + (exact-phrase + tag + type)×(1+semantic learned)
     + recency(temporal learned) + frecency + trust
     + ordinal boost (earliest/latest by event time)
     + feedback boost (helpful ids from similar past questions)
  → metadata filter enforcement
  → fallback: raw substring scan when index is empty
  → EvidenceProcessor: dedup(0.85 Jaccard) → trim by budget
  → SufficiencyScorer → [ABSTAIN] below threshold
  → SearchResponse: records + CCR formatted_context + sufficiency
  → access bump (version-preserving)

MAINTENANCE (cron tool or lifecycle scheduler)
  → accessibility decay (per-type half-life) → quota eviction (utility)
  → ConsolidationDaemon (tag-cluster + Jaccard≥0.5 → SUMMARY record,
     originals archived valid_until=now)
  → index rebuild refresh

VERSION GRAPH (backup manager)
  snapshots (ids + watermark + branch metadata)
  → branches (LMDB env.copy, atomic) → checkout
  → 3-way merge (source/target/newest/manual strategies)
  → content-level diff, JSONL export/import
```

## Background subsystems (started with the server)

| Subsystem | Role | Flag |
|---|---|---|
| Event bus + Sync scheduler | trust-gated agent→project commit (idempotent) | `sync_enabled` |
| Lifecycle scheduler | decay/quota/consolidation across tenants | `lifecycle_enabled` |
| File watcher | watch dirs → ingest chunks live | `ingest_watch_*` |
| Eager tenant warmer | open every tenant DB + index at startup | `eager_load_tenants` (default on) |

## The INDEX layer (model discovery)

- `admin action=index` (no tenant) → every tenant's memory count, type mix,
  top tags, time range — the "book index". It is **built live from LMDB on
  every call**: as soon as any store/ingest/delete lands, the next
  `admin action=index` sees it. With 50 tenants, this is how a model decides
  which tenant to target a search at.
- `admin action=index tenant_id=X` → full catalog of one tenant.
- `admin action=source_report source=<file>` → per-document memory
  decomposition (one document, exactly which chunks became which memories).
- The MCP server sends a one-page tenant INDEX in its protocol instructions
  at session initialize. That is a **session-start snapshot** (MCP has no
  mid-session push channel); the always-current INDEX is the
  `admin action=index` call — the system prompt tells models to re-read it
  whenever the landscape may have changed.

## Worked example — one file spread into the tenant

Ingesting `docs\example_architecture_spec.md` (18,165 bytes, markdown)
into a tenant named `demo`:

```
admin action=ingest  ->  30 memories created (ids 1-30)
admin action=source_report  ->  30 chunks, 17,653 chars
```

Each markdown heading section became one `document` record:

```
memory_id 1  | 802 chars  | tags: document_chunk, heading:…v1 → A persisting…, source:example_architecture_spec.md
memory_id 6  | 1068 chars | tags: document_chunk, heading:…v1 → 2. Attention…, source:…
memory_id 30 | 388 chars  | tags: document_chunk, heading:…v1 → 11. Non-goals, source:…
```

Physically: 30 rows in the LMDB `memories` sub-DB (`prototype record`: type
`document`, provenance `document`, confidence 1.0, accessibility 1.0,
`valid_from = created_at`). The `tag_index` sub-DB maps
`source:example_architecture_spec.md → [1..30]`,
`heading:…(each path) → [chunk ids]`, `document_chunk → [1..30]`. Because all
chunks share the `source:`/`document_chunk` tags at ≥0.5 tag-overlap, the
auto-linker wired every chunk's `related_memory_ids` to the others — the
viewer shows a single dense 30-node document cluster; search can hop
laterally from one section to a sibling heading. The Tantivy `terms` field
carries the normalized tokens of all 30 chunks so normal keyword search hits
sections directly; retrieval examples after this ingest (sufficiency score
shown): `"entity architecture specification"` → chunk 30 (0.48),
`"non-existent banana query"` → nothing standing (0.15 → `[ABSTAIN]`).
