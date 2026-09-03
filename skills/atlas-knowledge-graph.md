---
name: atlas-knowledge-graph
title: Atlas — persist and recall knowledge, state, graph, evidence and work sessions
audience: ai-agent
systems: [atlas]
version: 2
---

# Atlas — memory for agents

Atlas is the **state, knowledge and graph** Titan. It keeps what agents learn
and decide — packages of knowledge with evidence, typed graph layers, durable
work sessions, procedures that learn from their outcomes, scoped SQL and
analytics, blobs, provenance and cost — **entirely on the machine**, with an
append-only hash-chained audit trail. Every write is auditable; every read
comes back with the evidence behind it, not just a rank.

Use this skill when you need to remember something across sessions, ask
"what do we know about X", keep the state of a long piece of work, relate
things in a graph, or store the receipts of what ran and what it cost.

---

## Find Atlas by capability, never by name

Atlas announces these capability groups in the Titans registry:

```
admin  analysis  analytics  audit  blobs  capabilities  graph
procedures  provenance  search  sql  status  store  work
```

`discover_by_capability("store")` or `("search")` finds it. Do not look for a
Titan called `"atlas"` — product names change, capability strings are the
contract. The same strings are the `group` of every operation in
`contracts/atlas-operation-registry.json`, the file every surface is generated
from.

## Four ways in, one dispatch

| Surface | How | Notes |
|---|---|---|
| **MCP** (preferred for agents) | shared `titans` server, tools namespaced `titans_titan_atlas__atlas_search`; or direct `atlas mcp` over stdio | no port, no key; the installer wires Claude Code/Desktop, VS Code, Codex, Antigravity |
| **CLI** | `atlas call atlas <tool> '<json>'` | starts the daemon if it is not up; `atlas status` |
| **REST** | `http://127.0.0.1:7741` — `/openapi.json`, offline browser at `/docs` | loopback; bearer-token mode exists for programs |
| **Fast Contract IPC** | named pipe `titan_atlas`, canonical ids (`atlas.package.add`) | what other Titans use; MCP names are aliases of these ids |
| gRPC | `127.0.0.1:7742` | for programs that want it |

The **same dispatch table** serves all of them; a tool name works everywhere.
Canonical id ↔ MCP alias: `atlas.package.add` ↔ `atlas_add`,
`atlas.search` ↔ `atlas_search`, `atlas.work.session.create` ↔
`atlas_work_session_start`. The registry lists both.

**Stable JSON output** is a contract: responses never change shape between
surfaces or versions without a schema bump.

---

## Scope: everything lives in `(tenant, project)`

Every package, session, database, graph node and blob belongs to one tenant and
one project. Where the scope comes from is fixed per operation (`scope_policy`
in the registry):

| Policy | Meaning | Examples |
|---|---|---|
| `TopLevel` | `tenant` and `project` are arguments | `atlas_search`, `atlas_sql_query` |
| `Document` | read from the document you write | `atlas_add` (the package's own tenant/project) |
| `ExistingResource` | resolved from the id you point at | `atlas_get`, `atlas_delete` |
| `AdminGlobal` | machine-wide, no scope | `atlas_status`, `atlas_projects_list` |

`atlas_projects_list` shows every `(tenant, project)` this Atlas holds with
counts. Pick a stable pair per product/customer and keep using it.

---

## 1. Store and recall knowledge

### Store

A **KnowledgePackage** is markdown plus structure. The minimal one:

```json
{ "tool": "atlas_add", "args": { "package": {
  "schema_version": "atlas/1.0",
  "package_id": "decision-ledger-db",
  "tenant": "acme", "project": "alpha",
  "title": "Ledger database choice",
  "markdown": "We chose Postgres for the ledger because of range partitioning.",
  "tags": ["decision", "database"]
} } }
```

Optional and worth filling when you have them: `evidence` (blocks with the
source text the claim rests on), `entities`, `relations`, `graph_hints`
(feed the graph layers), `source` (where it came from), `assets` (blob refs),
`acl`. `package_version`, `fingerprint`, `created_at` are filled by Atlas.

The receipt is an `atlas://` **ref**, dedup-aware: adding the same content
twice returns the existing ref instead of a copy.

**Many at once:** `atlas_add_batch { packages: [...] }` commits the text index
and persists the vector snapshot **once** for the whole batch — far faster than
N single adds. Use it for imports.

### Recall

```json
{ "tool": "atlas_search", "args": { "tenant": "acme", "project": "alpha",
  "query": "why postgres", "top_k": 5, "include_evidence": true } }
```

Hybrid search (BM25 + vector + graph + evidence) with **explainable scores**:
each hit carries `package_ref`, `excerpt`, `tags` and a `rank_reason`
(`vector_score`, `graph_score`, `evidence_score`, `final_score`,
`explanation`, `deterministic`). Filters: `tags`, `media_types`,
`created_after/before`, `actor`. `atlas_plan_explain` shows the deterministic
filter plan without running it.

For a prompt, ask for a **context pack** instead of hits:

```json
{ "tool": "atlas_context", "args": { "tenant": "acme", "project": "alpha",
  "query": "ledger decisions", "max_tokens": 1500 } }
```

`sections` fit the budget; `excluded_due_to_budget` lists what did not — never
silently. `atlas_context_v2` puts package excerpts inline and every other
kind (work state, analysis, procedures, capabilities, graph nodes, SQL
catalog) **by ref with drill-down**; `atlas_search_resources` is the sectioned
search across all those kinds.

Read one thing: `atlas_get { id }` (id or `atlas://package/...` ref),
`atlas_markdown { id }` (body only), `atlas_evidence { id }`.

### Refs, not bytes

Large payloads travel as `atlas://tenant/project/...` references. Resolve any
ref — scope-qualified or legacy — with `atlas_ref_resolve { ref }`; an
ambiguous legacy ref is **refused**, not guessed. Never ask for bytes inline
when a ref exists.

### Truth-native: what changed, what conflicts

- `atlas_supersede { loser_id, package, reason? }` — the new package becomes
  Active and indexed, the old one Superseded and de-indexed. Use it when a
  fact is corrected; do not "add a newer version" beside the old one.
- `atlas_quarantine { id, reason?, winner_id? }` — hold an untrusted or
  conflicting package out of recall pending review; `atlas_restore { id }`
  brings it back.
- `atlas_conflicts { id }` — the transition records where a package was
  superseded or quarantined.
- `atlas_delete { id }` is a **soft** delete (reversible with
  `atlas_restore`); `atlas_hard_delete` is permanent and needs a grant (below).

Export/import a scope: `atlas_export { tenant?, project? }` →
`atlas_import { bundle }` (validated before persist).

---

## 2. Graph

Layers are typed: `knowledge`, `technical`, `dependency`, `semantic`, plus
custom ones. `atlas_graph_layers` lists them with visibility;
`atlas_graph_toggle { layer, visible }` hides a layer from every query and UI.

| Need | Tool |
|---|---|
| neighborhood of a node, N hops | `atlas_graph_neighbors { node_id, depth?, layers?, include_hidden? }` |
| the whole scoped graph (for rendering) | `atlas_graph_subgraph { tenant, project, layers?, limit? }` |
| provenance (ancestors) or impact (descendants) | `atlas_graph_lineage { node_id, direction: "ancestors"|"descendants", depth? }` |
| loops | `atlas_graph_cycles { tenant, project, max_cycles? }` |
| find by label | `atlas_graph_find { query, layer?, limit? }` |
| counts | `atlas_graph_stats` |
| RDF out | `atlas_graph_export_rdf { tenant, project, format: "turtle"|"ntriples" }` |
| manual notes | `atlas_graph_add_node { tenant, project, layer, label, node_kind?, props? }`, `atlas_graph_add_edge { tenant, project, layer, from, to, relation, weight? }` |

Manual nodes coexist with the automatic ones and are marked `origin=manual`.
Most graph content comes from the `entities`, `relations` and `graph_hints`
you put in packages — fill them and the graph builds itself.

---

## 3. Work sessions: durable state of long work

This is how an agent survives its own restart. One **context** per piece of
work; **sessions** inside it; a **monotonic event stream**; **checkpoints**
and **handoffs** as versioned artifacts.

Every work record is a **validated, versioned document**: it carries its own
`schema_version`, its scope (`tenant`, `project_id`, `context_id`), a
`canonical_payload` (your content, any JSON) and a `content_hash` — the blake3
hex of the canonical payload serialized as compact JSON — which Atlas verifies
on write. A native client library computes the hash for you; a hand-written
call must compute it too.

```json
{ "tool": "atlas_work_context_create", "args": { "context": {
  "schema_version": "atlas.work-context-record/1.0", "context_id": "ctx-migration-q4",
  "tenant": "acme", "project_id": "alpha", "objective": "Q4 ledger migration",
  "status": "active", "branch": "feature/ledger", "created_at": 1788470000, "updated_at": 1788470000 } } }

{ "tool": "atlas_work_session_start", "args": { "session": {
  "schema_version": "atlas.work-session-record/1.0", "session_id": "ses-0001",
  "tenant": "acme", "project_id": "alpha", "context_id": "ctx-migration-q4",
  "source": "claude-code", "status": "active",
  "canonical_payload": { "agent": "planner", "goal": "plan the migration" },
  "content_hash": "<blake3 hex of canonical_payload>",
  "last_sequence": 0, "acknowledged_sequence": 0, "started_at": 1788470000, "last_event_at": 1788470000 } } }

{ "tool": "atlas_work_event_batch", "args": { "batch": { "allow_gaps": false, "events": [ {
  "schema_version": "atlas.work-event-record/1.0", "event_id": "evt-0001",
  "tenant": "acme", "project_id": "alpha", "context_id": "ctx-migration-q4", "session_id": "ses-0001",
  "sequence": 1, "idempotency_key": "ses-0001:1",
  "canonical_payload": { "kind": "decision", "text": "Use range partitioning." },
  "content_hash": "<blake3 hex>", "occurred_at": 1788470010, "received_at": 1788470010 } ] } } }

{ "tool": "atlas_work_checkpoint_put", "args": { "checkpoint": {
  "schema_version": "atlas.work-checkpoint-record/1.0", "checkpoint_id": "ckp-0001",
  "tenant": "acme", "project_id": "alpha", "context_id": "ctx-migration-q4", "session_id": "ses-0001",
  "revision": 1, "canonical_payload": { "summary": "…", "open_items": [ "…" ] },
  "content_hash": "<blake3 hex>", "created_at": 1788470100 } } }

{ "tool": "atlas_work_handoff_put", "args": { "handoff": {
  "schema_version": "atlas.work-handoff-record/1.0", "handoff_id": "hof-0001",
  "tenant": "acme", "project_id": "alpha", "context_id": "ctx-migration-q4", "session_id": "ses-0001",
  "revision": 1, "source_checkpoint_ref": "atlas://…/ckp-0001",
  "canonical_payload": { "for": "next agent", "instructions": "…" },
  "content_hash": "<blake3 hex>", "created_at": 1788470200 } } }

{ "tool": "atlas_work_session_end", "args": { "session_id": "ses-0001", "status": "ended", "ended_at": 1788470300 } }
```

Rules that matter:

- Events are **monotonic and idempotent**: `sequence` only goes up, the
  `idempotency_key` makes a resend a no-op, and a gap is refused unless the
  batch says `allow_gaps: true`. Events are **redacted** before persist —
  never put secrets in a payload.
- A checkpoint or handoff is accepted only as **the next revision** for its
  context (`revision` = latest + 1); a stale revision is a `conflict`.
- `atlas_work_event_ack { session_id, sequence }` advances a cursor and only
  forward.
- Resume from `atlas_work_checkpoint_latest { tenant, project, context }` and
  `atlas_work_handoff_latest`; `atlas_work_decisions { project }` and
  `atlas_work_tasks_open { project }` are derived from the latest **verified**
  artifacts.
- A native client can bind its own id: `atlas_work_session_attach
  { source, native_session_id, session_id }` and later
  `atlas_work_session_resolve` → session id or `null`.
- A terminal session (`ended`/`interrupted`) rejects later events.
  `atlas_work_session_delete` is permanent and needs a grant.
- Timeline: `atlas_work_events { tenant, project, context, cursor?, limit? }`;
  lists: `atlas_work_contexts { project }`, `atlas_work_sessions`.

---

## 4. Procedures: runbooks that learn

`atlas_procedure_save { procedure }` stores a reusable procedure (steps,
preconditions, intent). Before doing something you have done before:

```json
{ "tool": "atlas_procedure_recall", "args": { "query": "rotate the signing key", "tenant": "acme", "project": "ops", "limit": 3 } }
```

Recall is intent-matched and **ranked by learned reliability**. After running
one, report `atlas_procedure_outcome { id, success }` — that is what moves the
ranking. `atlas_procedure_retire` takes one out of recall (history kept);
`atlas_procedure_restore` brings it back; `atlas_procedure_get`/`_list` read.

## 5. Capability registry: do not build what exists

Register a tool/skill/ability with `atlas_capability_register { capability }`
(its description is embedded). Before adding a new one:

```json
{ "tool": "atlas_capability_check", "args": { "name": "pdf-to-markdown", "description": "convert PDF pages to markdown with tables" } }
```

The verdict is `duplicate`, `overlaps` or `distinct`, with the matches. Treat
`duplicate` as "use the existing one".

## 6. Analysis runs and reports (titans.analysis/1.0)

Canonical analysis pipelines (Coeus, Cronus workers) persist their runs and
reports here as **immutable versions**: `atlas_analysis_run_create { run }`,
`atlas_analysis_run_update { run }` (exactly one version forward),
`atlas_analysis_run_get/list/latest`, `atlas_analysis_report_put { report }`,
`atlas_analysis_report_get/latest { tenant, project, kind }`,
`atlas_analysis_compare { older, newer }` (structural diff of two reports of
the same kind). `atlas_project_report { tenant, project }` is the human
overview: inventory, tag histogram, evidence totals, graph stats, top entities.

## 7. SQL and analytics, scoped

| Need | Tool |
|---|---|
| a database for this scope | `atlas_sql_create { tenant, project, display_name }` → `database_id`; `atlas_sql_list` |
| read | `atlas_sql_query { tenant, project, database_id, sql, params? }` (read-only) |
| write | `atlas_sql_execute { …, sql, params?, idempotency_key? }`; `atlas_sql_transaction { …, statements: [...] }` (all-or-nothing, journaled) |
| shape | `atlas_sql_schema`; `atlas_sql_backup` (consistent file beside the database) |
| drop | `atlas_sql_drop` — destructive, needs a grant |
| OLAP over the scope | `atlas_analytics_schema`, `atlas_analytics_query { sql, params? }`, `atlas_analytics_export { sql, format: ndjson\|csv\|parquet }`, `atlas_analytics_refresh` (drain the projection now) |

Pass `idempotency_key` on writes you might retry.

## 8. Blobs

Small payloads inline, large ones streamed:

- `atlas_blob_upload { tenant, project, owner_type, owner_id, media_type, file_path | bytes_base64 }` → `atlas://` ref. **Pass `file_path`** for anything big; the daemon streams it.
- Chunked on any surface: `atlas_blob_upload_begin` → `atlas_blob_upload_chunk { upload_id, data_base64 }`… → `atlas_blob_upload_finish { upload_id, owner_type, owner_id, media_type }` (hash, refcount, audit and ref in one transaction); `atlas_blob_upload_abort` drops the session.
- `atlas_blob_download { blob_hash, to_file_path? }` (stream to disk when large), `atlas_blob_download_range { start, len }`, `atlas_blob_meta`.
- `atlas_blob_delete_ref` releases **one owner reference**; the last one schedules GC. Destructive, needs a grant.

## 9. Provenance and cost

When a job ran somewhere (Cronus), keep the proof here:
`atlas_record_receipt { receipt }` stores the **execution receipt** (what ran,
seed/prompt, cost, output) as a provenance node linked to its output;
`atlas_record_usage { record }` appends a cost/usage record and returns the
running estimated+actual total for the project — the ledger providers consult
before spending. `atlas_trust_replay { actor?, op?, correlation_id?, kind?,
limit? }` replays the audit chain by cause.

## 10. Embeddings: one embedder per machine

`atlas_embed { text }` returns the vector **tagged with model and version**.
Atlas is the machine's ONE hosted embedder: never mix vectors from another
model or version into an Atlas index; a query embedded by a different version
is refused, not silently compared. `atlas_vectors_backfill` (admin) re-embeds
packages missing live vectors, idempotently.

---

## Destroying anything: preview → grant → confirm

Five operations are destructive with `destructive_policy: grant`:
`atlas_delete`, `atlas_hard_delete`, `atlas_work_session_delete`,
`atlas_sql_drop`, `atlas_blob_delete_ref`. Each needs a **one-use grant bound
to the exact resource**, issued by the preview and valid for **60 seconds**:

```json
{ "tool": "atlas_delete_preview", "args": { "target_tool": "atlas_hard_delete", "id": "atlas://package/acme/alpha/decision-ledger-db" } }
→ { "package_id": "…", "title": "…", "grant": "…", "expires_at": 1788470060 }
{ "tool": "atlas_hard_delete", "args": { "id": "atlas://package/acme/alpha/decision-ledger-db", "confirm": "<grant>" } }
```

`target_tool` defaults to `atlas_hard_delete`; for a database pass
`database_id`, for a blob `blob_hash` + owner, for a session the session
fields. The preview executes nothing and shows what would be destroyed.
`permission` and `mutability` are different axes: the preview is `read` but
requires `harddelete` permission — only someone entitled to destroy may even
ask what destruction looks like. Soft `atlas_delete` is reversible; prefer it.

## Errors: one set of codes on every surface

| Code | REST | Meaning for you |
|---|---|---|
| `validation`, `invalid_ref`, `config` | 400 | fix the request |
| `not_found` | 404 | wrong id/ref or scope |
| `conflict` | 409 | version/idempotency clash — read, then retry with the current state |
| `access_denied` | 403 | principal lacks the permission (read/write/harddelete/admin/export) |
| `timeout` | 408 | retry with backoff |
| `resource_exhausted` | 429 | **backpressure**: that scope's projection outbox is at its ceiling — wait, do not hammer |
| `storage`, `index`, `embedding`, `integrity`, `ambiguous_reference`, `internal` | 500 | report; `integrity` means the audit chain disagrees — stop and tell the operator |

## Rules that avoid rework

- **Refs, not bytes.** Store big content as blobs or files; pass refs.
- **Always scope.** Pick `(tenant, project)` once and use it everywhere; mixing scopes is how "I stored it but cannot find it" happens.
- **Batch bulk writes.** `atlas_add_batch` over a loop of `atlas_add`.
- **Correct facts with `supersede`,** not by adding a sibling. Search then shows one truth.
- **Report outcomes** of procedures; otherwise recall never learns.
- **Check capabilities** before inventing a tool.
- **Idempotency keys** on SQL writes and work events you might retry.
- **Do not poll** `atlas_world_model` or `atlas_status` in a loop; read them when you need a snapshot.
- **No cloud calls, ever.** Atlas is local; if you need a model, that is ProviderAdapter's job.

## What Atlas does NOT do

| Need | Go to |
|---|---|
| run heavy or long work durably | **Cronus** (`jobs`, `schedules`) — results archive back into Atlas |
| search code and files | **Coeus** (`search`, `analysis`) |
| decide whether an action is allowed | **Themis** (`policy`) |
| call a language model | **ProviderAdapter** (`llm.route`) |
| turn documents into knowledge | **uDoc** (containers) → then `atlas_add_batch` |

## Quick diagnostics

```bash
atlas status                                     # what is running, what this machine can do
atlas call atlas atlas_status '{}'               # packages, events, index counts, cache
atlas call atlas atlas_projects_list '{}'        # every (tenant, project) with counts
atlas call atlas atlas_audit_verify '{}'         # blake3 hash-chain intact?
atlas call atlas atlas_projection_status '{}'    # outbox backlog per scope (429s come from here)
atlas call atlas atlas_index_snapshot '{}'       # the read path never rebuilds an index
titans doctor                                    # the whole ecosystem
```

`atlas_world_model` is the live snapshot (packages, graph layers, Titans
presence, runtime); `atlas_storage_status` the Store Format v2 layout and
write-coordinator counters. Offline maintenance (`atlas_maintenance_backup`,
`_verify`, `_restore`) is refused while a daemon holds the store — stop it
first; `atlas maintenance` on the CLI does the same from a shell.

## Tool index (109)

| Group | Tools |
|---|---|
| store | `atlas_add`, `atlas_add_batch`, `atlas_get`, `atlas_markdown`, `atlas_evidence`, `atlas_delete`, `atlas_delete_preview`, `atlas_hard_delete`, `atlas_restore`, `atlas_supersede`, `atlas_quarantine`, `atlas_conflicts`, `atlas_export`, `atlas_import`, `atlas_ref_resolve` |
| search | `atlas_search`, `atlas_search_resources`, `atlas_context`, `atlas_context_v2`, `atlas_plan_explain`, `atlas_embed`, `atlas_project_report` |
| graph | `atlas_graph_layers`, `atlas_graph_toggle`, `atlas_graph_neighbors`, `atlas_graph_subgraph`, `atlas_graph_lineage`, `atlas_graph_cycles`, `atlas_graph_find`, `atlas_graph_stats`, `atlas_graph_export_rdf`, `atlas_graph_add_node`, `atlas_graph_add_edge` |
| work | `atlas_work_context_create`, `atlas_work_context_get`, `atlas_work_contexts`, `atlas_work_session_start`, `atlas_work_session_get`, `atlas_work_sessions`, `atlas_work_session_attach`, `atlas_work_session_resolve`, `atlas_work_session_end`, `atlas_work_session_delete`, `atlas_work_event_batch`, `atlas_work_event_ack`, `atlas_work_events`, `atlas_work_checkpoint_put`, `atlas_work_checkpoint_latest`, `atlas_work_handoff_put`, `atlas_work_handoff_latest`, `atlas_work_decisions`, `atlas_work_tasks_open` |
| procedures | `atlas_procedure_save`, `atlas_procedure_get`, `atlas_procedure_list`, `atlas_procedure_recall`, `atlas_procedure_outcome`, `atlas_procedure_retire`, `atlas_procedure_restore` |
| capabilities | `atlas_capability_register`, `atlas_capability_check`, `atlas_capability_list` |
| analysis | `atlas_analysis_run_create`, `atlas_analysis_run_update`, `atlas_analysis_run_get`, `atlas_analysis_run_list`, `atlas_analysis_run_latest`, `atlas_analysis_report_put`, `atlas_analysis_report_get`, `atlas_analysis_report_latest`, `atlas_analysis_compare` |
| sql | `atlas_sql_create`, `atlas_sql_list`, `atlas_sql_query`, `atlas_sql_execute`, `atlas_sql_transaction`, `atlas_sql_schema`, `atlas_sql_backup`, `atlas_sql_drop` |
| analytics | `atlas_analytics_schema`, `atlas_analytics_query`, `atlas_analytics_export`, `atlas_analytics_refresh` |
| blobs | `atlas_blob_upload`, `atlas_blob_upload_begin`, `atlas_blob_upload_chunk`, `atlas_blob_upload_finish`, `atlas_blob_upload_abort`, `atlas_blob_download`, `atlas_blob_download_range`, `atlas_blob_meta`, `atlas_blob_delete_ref` |
| provenance | `atlas_record_receipt`, `atlas_record_usage` |
| audit | `atlas_events`, `atlas_audit_verify`, `atlas_trust_replay`, `atlas_backup_events` |
| status | `atlas_status`, `atlas_storage_status`, `atlas_projection_status`, `atlas_index_snapshot`, `atlas_world_model`, `atlas_projects_list`, `atlas_cache` |
| admin | `atlas_maintenance_backup`, `atlas_maintenance_verify`, `atlas_maintenance_restore`, `atlas_vectors_backfill` |

Source of truth for arguments and responses: `contracts/atlas-operation-registry.json`
(`request_schema` / `response_schema` per operation) and `/openapi.json` on REST.
