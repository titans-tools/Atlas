# Atlas — Quickstart

## 1. Install (one line, no admin rights)

```powershell
irm https://github.com/titans-tools/titans-platform/releases/latest/download/Install-Atlas.ps1 | iex
```
```bash
curl -fsSL https://github.com/titans-tools/titans-platform/releases/latest/download/install-atlas.sh | sh
```

Check it: `titans doctor` and `atlas status`.

## 2. Store something worth remembering

```bash
atlas call atlas atlas_add '{"package":{"schema_version":"atlas/1.0","package_id":"note-1","tenant":"acme","project":"alpha","title":"Kickoff","markdown":"We chose Postgres for the ledger because of range partitioning."}}'
```

## 3. Ask, and get the evidence back

```bash
atlas call atlas atlas_search '{"tenant":"acme","project":"alpha","query":"why postgres"}'
```

The answer carries the matching package, a per-signal score breakdown
(lexical / vector / graph) and the evidence behind it.

## 4. Let your agents use it

The installer already wired the shared `titans` MCP server into the AI clients
on your machine (Claude Code, Claude Desktop, VS Code, Codex, Antigravity).
Ask your agent to call `atlas_search` — no ports, no keys.

## 5. Where to go next

- Everything is scoped by `(tenant, project)` — `atlas_projects_list` shows yours.
- Graph: `atlas_graph_neighbors` · SQL: `atlas_sql_query` · analytics:
  `atlas_analytics_query` · blobs: `atlas_blob_upload`.
- A browser configurator lives at `atlas config-ui`.
- Updates: `titans versions check` (signed catalogs; nothing changes without confirmation).
