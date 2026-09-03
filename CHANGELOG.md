# Changelog — Atlas

Full history begins at the first public stable line. Every release is signed
and described in the signed distribution catalog; the catalog is the authority.

## 0.2.8 — 2026-08-27
First public stable release on the `titans-tools` organization.

- Hybrid search (full text, vectors, graph, evidence) over knowledge packages,
  with per-signal score breakdown and evidence anchoring.
- Typed graph, durable work state (contexts, sessions, events, checkpoints),
  per-project SQLite, DuckDB analytics, content-addressed blobs, hash-chained
  audit — 109 operations, identical over MCP, REST, gRPC and the Titans Fast
  Contract.
- Operator controls: durable pause/resume and self-applied memory/priority
  limits (`titans control ...`).
- Certified end to end on pristine Windows and Linux machines against the
  public install path before publication.
