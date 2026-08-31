# MemStore_v2 — AI Model Setup

MemStore_v2 is a local, MCP-only memory substrate for AI agents. There is no
REST/HTTP API and nothing leaves your machine. It is **fully model-free**:
a Tantivy inverted index (pure-Python BM25 fallback) plus a heuristic
multi-signal reranker. No model downloads, no GPU, <100 ms query latency,
<1 s startup, eager tenant loading on connect.

## 1. Install

```bash
python install.py
```

The installer creates `.venv`, installs the full system, optionally writes
MCP client configuration, and runs a smoke test. No profiles to choose,
no ML extras to download — one install, everything included.

## 2. Connect your MCP client

Run the server over stdio:

```bash
python -m memstore --transport stdio
```

Or SSE (for clients that need it):

```bash
python -m memstore --transport sse --host 127.0.0.1 --port 8000
```

Register the stdio command in your client's `mcpServers` config:

```json
{
  "mcpServers": {
    "memstore": {
      "command": "C:\\MemStore\\venv\\Scripts\\python.exe",
      "args": ["-m", "memstore", "--transport", "stdio"]
    }
  }
}
```

## 3. System prompt

Copy the contents of `memstore/system_prompt.md` into your agent's system
prompt (or let your client include it with every prompt — it is short).
It teaches the agent to **look at the INDEX first** (`admin action=index`),
when to store, how to store (always include date & time in time-bound facts),
when/how to read, the feedback loop, and memory hygiene.

The MCP server also attaches a **live data INDEX** in its connection
instructions at session initialize, so the model already knows which tenants,
how many memories, and which top tags exist *before* the first prompt.

## 4. The 14 tools

| Tool | Purpose |
|---|---|
| `store` | Store a memory (auto dates, provenance inference, trust gate). |
| `update` | Patch a memory (preserves links and history). |
| `delete` | Delete by id (protected memories refuse). |
| `search` | Unified retrieval: modes, filters, budgets, abstention. |
| `drill_down` | Expand `[ref: mem_N]` into the full record. |
| `temporal_query` | `what_was_true_at`, `get_changes`, `get_timeline`, `recent`, `concept`. |
| `contradictions` | `detect`, `list`, `resolve` with persistence. |
| `version` | `snapshot`, `restore`, `branch`, `checkout`, `merge`, `diff`, `export`, `import`, `list`. |
| `task` | Working memory: `create`, `update`, `end`, `canvas` (Mermaid), `list`, `get`. |
| `feedback` | Trust nudge + sufficiency signal to the EMA learner. |
| `explore_graph` | BFS over related/contradicts edges. |
| `audit` | `log`, `list`, `summary`, `bundle` (support bundle). |
| `configure` | `get`, `set`, `list`, `reset`, `reload`, `doctor`. |
| `admin` | `status`, `list_tenants`, `stats`, `reset_cache`, `index`, `dump`, `export`, `import`, `metrics`, `ingest`, `maintenance`, `sync`, `source_report`. |

## 5. Common patterns

Store a constraint:

```json
{"tenant_id": "default", "content": "Never run migrations without a snapshot.",
 "memory_type": "constraint", "tags": ["db", "release"], "provenance": "user"}
```

Search with a token budget:

```json
{"tenant_id": "default", "query": "migration snapshot rule",
 "top_k": 5, "max_total_tokens": 800}
```

Resolve a contradiction:

```json
{"tenant_id": "default", "action": "resolve", "memory_id": 12,
 "target_memory_id": 34, "resolution": "The API moved to v2; v1 notes are obsolete."}
```

Export a snapshot of everything:

```json
{"tenant_id": "default", "action": "export"}
```

## 6. Understanding what you see

- The search response includes `formatted_context` (budget-aware compressed
  evidence with `[ref: mem_N]` anchors), `sufficiency_score`, and a
  `[ABSTAIN]` marker when the evidence is not enough — ask the user or
  store the gap instead of guessing.
- Search is ordinal-aware: ask "the first X" / "the latest X" and the
  engine boosts temporally earliest/latest events automatically (event
  times come from `valid_from`, extracted from dates in memory content).
- Architecture and the full database mapping are documented in
  `docs/architecture.md` and `docs/architecture.html`.
