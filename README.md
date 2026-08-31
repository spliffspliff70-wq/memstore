# MemStore v1.0

Fully model-free durable memory substrate for autonomous agents.

MemStore gives AI agents persistent, queryable memory with built-in temporal
truth, contradiction detection, versioned snapshots with branching and
three-way merge, multi-agent sync, trust scoring, online-learned retrieval
weights, and budget-aware evidence compression — over exactly 14 MCP tools.
No models, no downloads, eager startup, fully offline.

Architecture & database mapping: `docs/architecture.md` and
`docs/architecture.html`.

## Install (production)

**Recommended for users:** the release zip + `install.py` (no signatures
needed, no SmartScreen):

```powershell
Expand-Archive .\Downloads\memstore_v*_*.zip -DestinationPath C:\MemStore
python C:\MemStore\memstore_v2\install.py
```

The installer is interactive by default (numbered choices for install type,
tenant data dir, LMDB map size, license key) and fully scriptable
(`python install.py --type full --tenant-dir E:\data\tenants --yes`).
It writes a valid `config.yaml`, creates Start-menu shortcuts, writes
`uninstall.cmd` (keep-or-delete data prompt) — and it is **non-intrusive**:
no IDE/CLI configs are touched; it generates `MCP-CONFIGURATION.md` with
copy-paste connection snippets for every AI client detected so you wire
them yourself (or later via `clients-register --dry-run`, inspect, apply
explicitly).

**Developer path:** `python install.py --extras dev` (repo venv + smoke
test) or `pip install -e ".[dev]"` + `python -m memstore --transport stdio`.

Health check: `python -m memstore doctor`
Release zip: `python -m memstore build`

## Updating an existing install

Run the NEW zip's installer against the existing install — no manual
extract-over needed:

```powershell
python C:\Downloads\memstore_v2\install.py --update C:\Downloads\memstore_v1.0.1.zip --dir C:\MemStore\memstore_v2
```

Conflict-safe by design: files you modified are never overwritten (the new
version lands next to yours as `name.new-<version>` with a merge report),
`config.yaml`, the venv, logs and all tenant data are untouched, every
replaced file is backed up to `backups/pre-update-<timestamp>.zip` first,
and re-running the same update is a no-op.

## Two folders: program vs. memories

The install folder holds **only the program**. Your memories live in the
**tenant data dir** you choose during setup (Browse… button in the wizard,
default `%LOCALAPPDATA%\MemStore\data\tenants`):

- `tenant_<id>.db` — one LMDB database per tenant, created lazily on first
  store; `tenant_<id>_<branch>.db` for branched histories.
- `backups/`, `weights/`, `feedback/` — snapshots, EMA weights, feedback log.

Every client connected by the installer gets its `memstore` MCP entry
**pinned to that tenant data dir** via `env = { MEMSTORE_TENANT_DIR = ... }`,
so Kimi CLI, OpenCode, Claude, Cursor etc. all share one memory root even
when the program sits on C:\ and projects live on E:\. Tenants follow
**project names, not tools**: `autosave add E:\proj\B` → tenant `b`
(`tenant_b.db`), regardless of which client you use in that folder. Full
walkthrough: `docs/user_install.md`.

## Memory types & half-lives

Each type has a half-life — the time in which an untouched memory's
accessibility halves; accessing a memory refreshes it, decay is never
deletion, and `constraint` (0) never decays. Curated default set: `semantic`,
`fact`, `finding`, `decision`, `constraint`, `self`, `skill` (tool/workflow
competence), `procedural`, `code` (snippets/APIs), `plan`, `goal`,
`progress`, `question`, `hypothesis`, `episodic`, `lesson`, `state`,
`observation`, `snapshot`, `document`, `summary`, `symbolic`. Legacy types
(`instruction`, `email`, `log`) remain valid for stored history. Half-lives
and colors: `memstore/core/schema.py`.

## Documentation

- `docs/model_install.md` — how AI models/agents should use MemStore.
- `docs/user_install.md` — non-technical install and daily usage.
- `memstore/system_prompt.md` — copy-paste prompt for agents (<8k tokens).
- `docs/architecture.md` (+ the HTML version under `docs/architecture.html`) — engine internals.

## Architecture

- **Storage:** LMDB with named sub-databases, temporal + tag indices,
  graph adjacency.
- **Primary index:** Tantivy full-text over content, tags, memory type,
  provenance. Pure-Python BM25 fallback when unavailable.
- **Retrieval:** non-neural-first `V2Retriever` with metadata filtering,
  graph expansion, online-learned weight blends, feedback boosting, and
  sufficiency-aware abstention.
- **Trust:** provenance/consistency/anomaly trust scoring; trust-gated
  commits refuse contradictions of high-trust memory.
- **Maintenance:** lifecycle scheduler applies accessibility decay, quota
  enforcement, and consolidation summaries.
- **Live INDEX:** the MCP server describes all tenants in its connection
  instructions; `admin action=index` / `source_report` expose the full map.
- **No neural stack:** measured dense-fallback quality matched the lexical
  index, so the dependency was dropped — the product needs no models, CPU or GPU.

## Benchmarks & external audit

`benchmarks/` contains four independent evaluation suites, all public and
reproducible:

- `compare_v1_v2.py` — head-to-head with the dense v1 stack on a curated
  52-query golden set from real tenants. Latest: **v2 R@10 = 1.000** (0
  misses) vs v1 0.923; median query latency **23 ms vs 626 ms**.
- `locomo_benchmark.py` — official ACL 2024 LoCoMo run (1,986 QA), plus
  BEAM, adversarial contradiction (F1 1.000), scale/stress, and ingestion
  throughput harnesses. Scores and methodology in `benchmarks/README.md`.

```bash
python benchmarks/build_golden_set.py
python benchmarks/compare_v1_v2.py
python benchmarks/locomo_benchmark.py
python benchmarks/uat_model_audit.py
```

## License

MemStore is distributed under the **MemStore Proprietary License
(MPL-1.0)** — see `LICENSE`. It is source-available for inspection and
debugging, but it is not open-source: you may use it internally with a
valid license, but you may not redistribute it, resell it, sublicense it,
or build a competing paid product from it. Update checks and license
verification are local-only.
