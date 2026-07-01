# Positioning: Agent-Memory / RAG Substrate

Comparison of km against CozoDB, Oxigraph, and KùzuDB, strictly from the lens of "memory store for LLM agents / grounded retrieval." Status facts checked 2026-06-12.

## What the niche requires

- **Embedded, in-process, local-first** — runs inside the agent process, no server.
- **Hybrid retrieval** — vector search for entry points + exact graph traversal (GraphRAG core loop).
- **Guarded writes** — LLM-extracted facts are unreliable; the store should reject malformed writes deterministically.
- **Runtime schema evolution** — agents discover new relation types mid-session; schema changes must go through the same (validated) write path, not DDL.
- **Provenance per fact** — source, confidence, timestamp on individual edges; memory without provenance can't be audited or decayed.
- **Inspectability/export** — a human or another model must be able to read the whole memory.
- **Longevity** — agent memory is durable state; the store must outlive framework churn.

## Landscape status (verified)

- **KùzuDB**: archived by Kùzu Inc. on GitHub 2025-10-10; no longer actively supported. Community forks exist (bighorn by Kineviz; Ladybug).
- **CozoDB**: last feature release line v0.7 (HNSW added in v0.6); release cadence stalled since ~2023–24, effectively single-maintainer; issues still trickle in. Tagline is literally "the hippocampus for AI" — same niche.
- **Oxigraph**: actively maintained (releases through 2026; RDF 1.2 / SPARQL 1.2 behind feature flags; transactions). No vector index.
- Net: the embedded-graph-for-AI niche lost its leader (Kùzu) and its closest km-analogue is dormant (Cozo). Oxigraph is the only healthy option and lacks the vector half. The lane is more open than when the roadmap was written — and the niche's demonstrated failure mode is sustainability, not technology.

## Per-system comparison

### CozoDB (embedded Datalog, relational-graph-vector)

| | |
|---|---|
| Gives | In-process Rust; Datalog with recursion; built-in HNSW vector search in queries (true one-engine hybrid retrieval); time-travel queries; mem/SQLite/RocksDB backends; Python/JS/Java bindings |
| km adds | Schema-as-data (agent can read/extend schema via the same API); closed-definition validation on every write; qualifier typed/constrained at the definition; instance-vs-schema target levels |
| km lacks | Recursive query language; time travel; non-Rust bindings; backend choice; years of testing |
| Build-on verdict | Technically the best substrate (only one with integrated vectors) but maintenance risk makes it a foundation of sand; fork-and-own would be the only safe mode |

### Oxigraph (embedded RDF + SPARQL)

| | |
|---|---|
| Gives | Active maintenance; standards interop (Turtle/SPARQL — instant ecosystem, LLMs already write SPARQL); transactions; RDF 1.2 triple terms ≈ statement qualifiers; named graphs usable for per-source/per-session memory partitions |
| km adds | Vector layer (Oxigraph has none — sidecar required, which km's embed/HNSW design already is); validation (RDF needs external SHACL; nothing rejects a bad write by default); a far smaller model for agents to target than full RDF/SPARQL |
| km lacks | SPARQL; standards; maturity; Python bindings (pyoxigraph) |
| Build-on verdict | The only viable build-on today: km-core as a typed/validating layer over Oxigraph + the existing embedding sidecar. Costs: impedance mismatch (sets/qualifiers → RDF encoding), SPARQL dependency |

### KùzuDB (embedded property graph, Cypher) — archived

| | |
|---|---|
| Gave | Cypher; vector + full-text indices; columnar analytics performance; WASM; edge properties richer than km's single qualifier |
| Status | Archived 2025-10; forks (bighorn, Ladybug) young and unproven |
| Relevance now | Not a build-on candidate. Two lessons: (1) its archival created the vacancy km would fill; (2) its users were burned — "who maintains this in 3 years" will be the first adoption question km faces |

## Where km is genuinely differentiated for this niche

1. **Validated writes** — closed definitions reject hallucinated/malformed facts at the store boundary. None of the three do this natively. For agent memory this is the headline feature: the store is the type-checker for the LLM's output.
2. **Schema-as-data = agent-evolvable schema** — an agent can query what kinds of facts are storable and propose new sets/definitions through the same entity/relationship API, still under validation. Cozo/Kùzu need DDL; RDF allows it but unvalidated.
3. **Pure-store/processing split** — embeddings and LLM layers are regenerable sidecars; memory survives model swaps losslessly. (Good engineering; Cozo partially shares it.)

## Gaps km must close to be credible here (priority order)

1. **Provenance**: one qualifier per edge cannot carry source + confidence + timestamp. Either allow multiple qualifiers or make edge reification cheap and idiomatic; agent memory is unshippable without this.
2. **Python bindings**: agents are written in Python; a Rust-only crate is invisible to the target user.
3. **MCP server**: expose the store as an MCP memory server (store_fact / recall / traverse / validate tools). Cheapest possible distribution into every agent framework at once; also forces the API to be LLM-ergonomic.
4. **Concurrency**: multiple agents / agent + human writing simultaneously; current store is single-writer in-process.
5. **Memory lifecycle**: timestamps + decay/archival policy (even just "supersedes" edges); recall scoring that mixes vector similarity with recency.

## Verdict

- **Lane is open**: post-Kùzu, no actively maintained embedded store offers hybrid symbolic+vector retrieval; none ever offered validated writes.
- **Recommended path**: continue build-under (own store) for the core; the differentiators (validation, schema-as-data) live below the query layer and are awkward to retrofit onto Oxigraph, and Cozo is unmaintained. Re-evaluate Oxigraph-as-backend only if persistence/scale work (roadmap §3) stalls.
- **Reframed pitch**: not "a knowledge graph" but "typed, validated memory for agents — the store that refuses bad facts."
- **Sustainability answer required at launch**: permissive license, small surface, plain-Rust no-exotic-deps, documented format — "easy to fork" as an explicit feature, learning from Kùzu.

## References

- Kùzu archival: theregister.com/2025/10/14/kuzudb_abandoned · news.ycombinator.com/item?id=45560036
- Forks: gdotv.com/blog/weekly-edge-adieu-kuzu-state-of-the-graph-17-october-2025
- Cozo status: github.com/cozodb/cozo · cozodb.org
- Oxigraph status: github.com/oxigraph/oxigraph (CHANGELOG) · crates.io/crates/oxigraph
