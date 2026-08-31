# MemStore — Installation & Daily Use (No Coding Required)

MemStore gives your AI assistant a permanent, private memory that lives on
your computer. Nothing is sent to the cloud, no models are downloaded, and
it works fully offline.

## How the whole system works (read this once)

Stages of a memory, from your chat to your AI's next answer:

```
   YOU + YOUR IDE/CLI of the day
   Kimi CLI   OpenCode   Cursor   ...   every client has a memstore
                                          MCP entry (you, hand-written
                                          or installer-added) pointing
   ┌─── processes per session ───┐      at the same runtime.
   │  memstore server (spawned)  │
   │  reads MEMSTORE_TENANT_DIR  │
   │  env -> one shared data dir │
   └──────────────┬──────────────┘
                  │  store / search / update / delete / temporal / version
                  ▼
        ONE tenant data dir (you chose it):
        tenant_alpha.db   project A     every client, same project => same tenant
        tenant_beta.db    project B     no client silos.  One project folder
        tenant_gamma.db   project C     name => one tenant id, all tools the same.
        tenant_alpha_index\ ...         search index per tenant
        backups/  weights/  feedback/   extras
        autosave.yaml (opt-in watches)

        also streaming into them (only if you opt in):
           autosave sweeps your project's transcript files
           (.kimi/.opencode/sessions/*.jsonl etc.)
             -> episodic memories in the project's tenant
             (hash-deduped, tagged autosave:<client>, event-dated)
```

Two things to hold onto from that picture:

- **The program and the memories live in different folders.** The install
  dir (`C:\MemStore` etc.) only holds the program. Your memories are one
  LMDB file per project at `<your chosen tenant data dir>/tenant_<project>.db`.
- **Tenants follow projects, not tools.** Folder name = tenant id.
  Kimi, opencode, antigravity, codex — any client on the same project
  folder reads and writes the same tenant file, appending the same
  timeline. Switch tools mid-project and nothing migrates: the tenant was
  never the tool's.

## The two-folder model (read this once, it explains everything)

MemStore keeps two completely separate places:

| Folder | Contents | Example |
|---|---|---|
| **Install folder** | The program itself: runtime, engine, docs, `config.yaml`, logs | `C:\MemStore` |
| **Tenant data dir** | Your memories: one database per tenant (`tenant_<name>.db`), plus backups, weights, feedback | `E:\memstore-data\tenants` (you choose it) |

You pick both during setup. The wizard shows a **Browse…** button for the
tenant data dir and a Browse button on the destination page — put the
**program on your system drive** and the **memories on the drive where your
projects live**, so they survive reinstalls and stay close to your work.

`config.yaml` in the install folder records what you chose, and every AI
client that connects is pinned to that same tenant data dir automatically.

| Folder | Contents | Example |
|---|---|---|
| **Install folder** | The program itself: runtime, engine, docs, `config.yaml`, logs | `C:\MemStore` |
| **Tenant data dir** | Your memories: one database per tenant (`tenant_<name>.db`), plus backups, weights, feedback | `E:\memstore-data\tenants` (you choose it) |

You pick both during setup. The wizard shows a **Browse…** button for the
tenant data dir and a Browse button on the destination page — put the
**program on your system drive** and the **memories on the drive where your
projects live**, so they survive reinstalls and stay close to your work.

`config.yaml` in the install folder records what you chose, and every AI
client that connects is pinned to that same tenant data dir automatically.

## Install (Windows)

1. Download the release zip (`memstore_v<version>_<date>.zip`) from your
   Lemon Squeezy receipt/download email and unzip it, e.g.:
   `Expand-Archive .\memstore_v*_*.zip -DestinationPath C:\MemStore`
2. Run the installer: `python C:\MemStore\memstore_v2\install.py`
   (requires Python 3.11+ — it builds MemStore's own portable venv; nothing
   is ever installed into your system Python).
3. Choose **Full** (engine + docs + shortcuts) or **Compact** (engine only).
4. Choose the **install folder** (the program lives here, e.g. `C:\MemStore`).
5. Paste your **Lemon Squeezy purchase key** — the UUID from your receipt
   email (required; the install aborts on an invalid one). The key is
   activated **once** against the vendor at install, then works offline
   forever.
6. Choose the **tenant data dir** (browse-style prompt — this is where your
   memories will live; default
   `%LOCALAPPDATA%\MemStore\data\tenants`) and pick the LMDB map size
   (250/500/1000 MB, or custom).
7. Finish. The installer writes `config.yaml`, generates
   `MCP-CONFIGURATION.md` with copy-paste connection snippets for every AI
   tool it detected on your machine (Kimi CLI, Claude Desktop/Code, Cursor,
   Codex, Gemini CLI, OpenCode, OpenClaw, Hermes, CommandCode CLI,
   Antigravity, Kiro, Cline, Kilo Code, Roo Code, Continue, Windsurf, Zed,
   VS Code Copilot, Amp, Goose, and more), creates start-menu shortcuts and
   `uninstall.cmd`, then runs a health check + smoke test. **No IDE/CLI
   configs are touched** — you connect clients yourself from
   `MCP-CONFIGURATION.md`, or later via
   `python -m memstore clients-register --dry-run` (preview) and
   `clients-register` (apply, with `.bak` backups).

Everything is scriptable too:

```powershell
python install.py --dir C:\MemStore --type full `
    --tenant-dir E:\memstore-data\tenants `
    --license-key <your-lemonsqueezy-key> --yes
```

## What each connected client gets

When you connect *Kimi CLI* (via `clients-register`, with `.bak` backup
first), MemStore adds one entry to `~/.kimi/config.toml`:

```toml
[mcp_servers.memstore]
command = "C:\MemStore\venv\Scripts\python.exe"
args = ["-m", "memstore", "--transport", "stdio"]
env = { MEMSTORE_TENANT_DIR = "E:\memstore-data\tenants" }
```

The `MEMSTORE_TENANT_DIR` env var is what makes *all* your tools share the
one tenant data dir you chose — even though the program lives on C:\ and
your projects live on E:\.

## Example: projects A and B in Kimi CLI, C and D in OpenCode

Say you install MemStore to `C:\MemStore`, chose `E:\memstore-data\tenants`
as the tenant data dir, and connected Kimi CLI and OpenCode (via
`clients-register` or hand-pasted snippets). Here is exactly what happens:

- **C:\MemStore** gets only the program (runtime, engine, config.yaml,
  docs, logs). No project data ever goes there.
- **No tenants exist yet.** Tenants are created lazily — a tenant appears
  in the tenant data dir the first time a memory is stored for it.
- While working in project A inside Kimi CLI, the assistant calls
  `store` with tenant `projecta` → `E:\memstore-data\tenants\tenant_projecta.db`
  is created. Project B via Kimi → `tenant_projectb.db`. Projects C and D
  via OpenCode → `tenant_projectc.db`, `tenant_projectd.db`.
- **Autosave** (see below) is what ties a project folder to a tenant
  automatically: `memstore autosave add E:\proj\B --platform kimi_cli`
  maps *exactly* to tenant `B` — every transcript turn captured while
  working in `E:\proj\B` lands in `E:\memstore-data\tenants\tenant_B.db`,
  tagged `autosave:kimi_cli` and `project:b`.
- The tenant name is derived from the **project folder name** (lowercased,
  spaces → `_`): `E:\proj\My App` → tenant `my_app`. You can also set the
  tenant explicitly with `--tenant` if you prefer a custom id.
- One project, one tenant — regardless of which client you used in it.
  Kimi CLI and OpenCode in the same project folder both write the same
  tenant, so memory follows the project, not the tool.

## What is "half-life" and why should you care?

Every memory type decays with age so that *fresh, relevant* memories rank
higher in searches — like a human remembering recent events more easily.
Each type has a **half-life**: the time in which an untouched memory's
`accessibility_score` halves (50% decay).

- `episodic` (chat turns) — half-life **1 day**: conversation details fade fast.
- `plan`, `progress`, `question` — hours to days: work-in-progress is volatile.
- `procedural`, `self`, `skill`, `code`, `finding`, `decision` — **7–60 days**.
- `semantic`, `fact`, `lesson`, `document`, `snapshot`, `summary` — **30–90 days**.
- `constraint` — half-life **never** (0): hard rules stay at full strength forever.

Decay is gradual, not deletion: old memories never vanish, they just score
lower and get retrieved only when clearly relevant. **Accessing a memory
refreshes it** — a fact you (or your assistant) keep using never grows
stale. A background maintenance pass applies decay; the retention quota
only evicts already-decayed, low-value records.

## Capture your conversations automatically (opt-in)

If you want MemStore to keep a project-level memory of your chat
transcripts, run once per project:

```powershell
python -m memstore autosave add E:\my-project --platform claude_code
python -m memstore autosave list
# run the watcher once, or as a background daemon:
python -m memstore autosave run --once
python -m memstore autosave run --interval 300
```

Each transcript turn becomes an episodic memory in the tenant named after
your project folder (`E:\my-project` → tenant `my-project`), tagged
`autosave:<platform>` and `project:<name>`, and deduplicated by content
hash (re-running the sweep never writes duplicates). This is **opt-in**:
nothing watches anything until you add a project. Removing a project only
stops watching it; previously saved memories remain.

Available platforms: `claude_code`, `kimi_cli`, `cursor`, `opencode`,
`openclaw`, `hermes`, `codex`, `commandcode`, `antigravity`, `gemini`,
`cline`, and a generic fallback that scans common `sessions/`,
`conversations/`, `transcripts/`, and `*.jsonl` locations in the project
directory.

**Native session stores (v5.3).** Several platforms are ingested straight
from their real session store wherever it lives on disk — no project
dot-dirs required: `codex` (`~/.codex/sessions` rollout JSONL), `openclaw`
(`~/.openclaw/agents/*` trajectory JSONL), `opencode` (`opencode.db`
SQLite), `cursor` (`~/.cursor/chats` + `acp-sessions` SQLite), `gemini`
(`~/.gemini/tmp/*/chats` JSONL), and `cline` (`~/.cline/data/sessions`
JSON). Point the watcher at any directory (even your home dir) and these
adapters pick up their tool's full history; the project path only filters
by recorded cwd where the store carries one.

## Everyday use

Talk to your AI assistant normally. After setup it can remember things for
you. Try:

- "Remember that our staging API is at https://staging.example.com."
- "What database did we decide to use?"
- "Summarize what changed this week."
- "Take a snapshot before we import these files."

Useful commands you can ask the assistant to run for you:

- **Check health** — `python -m memstore doctor`
- **Back up your memory** — `python -m memstore export --tenant default`
- **Look at your memory graph** — `python -m memstore viewer --tenant default`

## Connecting extra clients later

Detected again on demand, or registered explicitly:

```powershell
python -m memstore clients-register            # all detected clients
python -m memstore clients-register --dry-run  # preview what would change (nothing written)
python -m memstore clients-register --kimi --opencode
python -m memstore clients-register --force    # repoint ALL entries to THIS install (dangerous if stale)
python -m memstore clients-register --project E:\proj\A   # project-scoped .mcp.json
```

Every existing config file gets a `.bak` backup; re-running never creates
duplicates; auto-repair only rewrites entries whose interpreter is missing
or v1-era. **Always verify with `--dry-run` first.**

### Manual connection (always safe — copy-paste per client)

If you prefer not to let the installer touch configs, these are the exact
snippets. Adjust the `command` only (path to the bundled Python) and the
`MEMSTORE_TENANT_DIR` pin (your memories):

```toml
# kimi CLI / codex CLI
# ~/.kimi/config.toml  or  ~/.codex/config.toml
[mcp_servers.memstore]
command = "C:\\MemStore\\venv\\Scripts\\python.exe"
args = ["-m", "memstore", "--transport", "stdio"]
env = { MEMSTORE_TENANT_DIR = "C:\\MemStore\\data\\tenants" }
```

```json
// claude_desktop / claude_code / cursor / cline / continue / gemini_cli / amp /
// windsurf / commandcode — their mcpServers JSON block:
{
  "mcpServers": {
    "memstore": {
      "command": "C:\\MemStore\\venv\\Scripts\\python.exe",
      "args": ["-m", "memstore", "--transport", "stdio"],
      "env": { "MEMSTORE_TENANT_DIR": "C:\\MemStore\\data\\tenants" }
    }
  }
}
```

```json
// opencode — its "mcp" object (note: "environment", NOT "env")
{
  "mcp": {
    "memstore": {
      "type": "local",
      "command": ["C:\\MemStore\\venv\\Scripts\\python.exe", "-m", "memstore", "--transport", "stdio"],
      "enabled": true,
      "environment": { "MEMSTORE_TENANT_DIR": "C:\\MemStore\\data\\tenants" }
    }
  }
}
```

```json
// openclaw — mcp.servers nesting
{
  "mcp": {
    "servers": {
      "memstore": {
        "command": "C:\\MemStore\\venv\\Scripts\\python.exe",
        "args": ["-m", "memstore", "--transport", "stdio"],
        "env": { "MEMSTORE_TENANT_DIR": "C:\\MemStore\\data\\tenants" },
        "enabled": true
      }
    }
  }
}
```

```yaml
# hermes — config.yaml, mcp_servers block
mcp_servers:
  memstore:
    command: C:\MemStore\venv\Scripts\python.exe
    args: [-m, memstore, --transport, stdio]
    env:
      MEMSTORE_TENANT_DIR: C:\MemStore\data\tenants
    enabled: true
```

VS Code-family (cursor user settings, antigravity, kiro, watsonx) uses the
top-level dotted key `"mcp.servers"` with the same entry as the JSON block
above.

After adding anything: start the client, ask it to run
`configure action=doctor`, and confirm it reports Healthy. If a client
ignores the entry (some read their config only on launch), restart it.

## Troubleshooting

| Problem | Fix |
|---|---|
| "python not found" | Use the bundled runtime: `"C:\MemStore\venv\Scripts\python.exe" -m memstore ...` |
| Server won't start | Run `python -m memstore doctor` and share the output. |
| Everything looks empty | Check the tenant id; run `admin action=list_tenants`. |
| Old facts keep surfacing | Ask the assistant to search and `delete`/`update` them, or run maintenance. |
| Memories not in my tenant dir | Check `config.yaml` → `tenant_dir`, and that the client entry carries `env = { MEMSTORE_TENANT_DIR = ... }`. |
| Client config was overwritten | Restore `<file>.bak` next to it. |

## Updating

Download the new zip, then run the NEW zip's installer against your existing
install:

```powershell
python C:\Downloads\memstore_v2\install.py --update C:\Downloads\memstore_v1.0.1.zip --dir C:\MemStore\memstore_v2
```

Updates are conflict-safe: files you modified are kept (the new version is
written next to them as `<name>.new-<version>` with a merge report),
`config.yaml` and all your memories are never touched, and every replaced
file is backed up to `backups/pre-update-<timestamp>.zip` first. Re-running
the same update is a no-op.

## Uninstalling

Use "MemStore — Uninstall" from the start menu. You are asked whether to
keep or delete your memory data; the program itself (including the bundled
runtime) is removed completely.
