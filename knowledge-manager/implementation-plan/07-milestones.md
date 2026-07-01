# Milestones

Each phase ships a compiling crate with tests; later phases add features without editing the core.

Status: complete. Phases 0–4 + durable/scale backends done (`cargo test --workspace --all-features`). Embeddings (L1) and the LLM processor (L2) are now the edge crates `km-edge-enrich-embed`/`km-edge-enrich-llms`, not km-core (`design/decisions.md`). Local inference out of scope (CPU-only target).

## Phase 0 — Model + in-memory store + bootstrap  ✅
- `id`, `literal`, `model` types; `MemStore` + indexes; `bootstrap::seed`.
- Accept: seed an empty store; round-trip serialize an entity; name/reverse/by_def indexes correct after put/delete.

## Phase 1 — Schema decode + resolution + validation  ✅
- `schema` views, `resolve::effective_sets`, `validate_*`.
- Load the full `example.md` dataset (Instrument/Transmitter/Controller/SignalConverter/ProcessSignals + 3 points + groups).
- Accept: effective sets match design (§"Effective sets"); `validate_all` = 0 violations on the example; deliberately corrupt cases produce each `Violation` variant.

## Phase 2 — Query, predicates, views + persistence + CLI  ✅ (bincode snapshot + redb backend)
- `Graph` API, `predicate` layer, `view` projections; `redb` backend behind `persist`; `km-cli`.
- Accept: schema/data/tree/by-type/neighborhood views render the example correctly (incl. `by-type Instrument` returning all three via build-on, and incoming `Data flow "PV"`); computed `>`/`<`/range over ranges; persist→reopen→identical dump; CLI drives load/validate/view/query.

## Phase 3 — Level 1 embeddings (crate `km-edge-enrich-embed`)  ✅ (HashEmbedder; HNSW added, fastembed optional)
- `Embedder`/`VectorIndex` sidecar; `ChangeSink` recompute on put.
- Accept: lexical-overlap query retrieves the right entity (true synonymy needs a learned model — drop-in via `Embedder`); hybrid query (`"flow instrument"` ∩ `located-in=Unit-02`) returns the three instruments, not the Flow value-entity; sidecar regenerable + deterministic, core snapshot unchanged.

## Phase 4 — Level 2 LLM processor (crate `km-edge-enrich-llms`)  ✅ (MockLlm tested; AnthropicLlm via `llm-http`)
- `Processor::populate` (extraction + embed-based dedup) and `answer`/`graphrag_context` (GraphRAG); `LlmClient` transport.
- Accept: ingest a doc → one new entity created, near-duplicate ("flow" vs "Flow") merged via embeddings, result validates clean; GraphRAG context for a NL question contains the connected edge (`02FIC100 Data flow [OP] -> 02FY100`); answer prompt grounded in that context. Real transport compiles behind `llm-http`.

## Durable storage — redb backend (feature `persist`)  ✅
- `RedbStore`: embedded ACID, CPU-only, single file; same `Store` trait; entity table is truth, indexes rebuilt on open; id counter persisted.
- Accept: full example round-trips through close/reopen and validates clean; put/delete maintains reverse index; ids never reused across reopen.

## Scale retrieval — HNSW index (km-edge-enrich-embed feature `hnsw`)  ✅
- `HnswIndex`: approximate NN (hnsw_rs, CPU), built from the `VectorIndex` sidecar; rebuild-on-drift.
- Accept: brute-force top-1 is recalled within the ANN top-k on the example; recalls a known entity; dimension/`k=0` guards.

## Phase 5 — Level 3 local inference  (out of scope)
- Dropped by decision: target is a GPU-less machine; CPU embeddings (`fastembed`/ONNX) + a remote `LlmClient` cover the need. The `Embedder`/`LlmClient` traits remain the swap points if revisited.

## Cross-cutting
- Tests: unit per module + one integration test that is the `example.md` dataset.
- Invariants as `debug_assert`/tests: no stored `>`/`<` edge; no vector in `entities`; every relationship resolves to a def under `validate`.
- Bench `effective_sets` + vector search at N=100k before optimizing (HNSW threshold).
