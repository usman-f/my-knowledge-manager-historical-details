# Overview — Crate Layout & Dependencies

## Workspace
```
knowledge-manager/
  Cargo.toml                 # [workspace]
  crates/
    km-core/                 # lib: pure store — model, store, bootstrap, resolution, validation, query, views
    km-cli/                  # bin: thin CLI over km-core (load/query/dump)
    km-edge-enrich-embed/    # edge over core: Level 1 embeddings + vector/hybrid retrieval (+ HNSW)
    km-edge-enrich-llms/     # edge over core: Level 2 LLM processor (extraction + GraphRAG); future home of claims
```
Processing was moved out of km-core into the two `km-edge-*` crates (2026-06-24, `design/decisions.md`).
Crate naming convention: `km-edge-[stage]-[name]` (lifecycle stage ∈ `load` | `enrich` | `use`).

## km-core module tree (pure infrastructure)
```
src/
  lib.rs            # re-exports, prelude
  id.rs             # Id, IdGen
  literal.rs        # Literal, LiteralType
  model.rs          # Entity, Relationship, Target, DefRef
  store/
    mod.rs          # Store trait, StoreError, MemStore, Indexes (HashMap backend)
    redb.rs         # persistent backend (feature = "persist")
  bootstrap.rs      # meta set seed
  schema.rs         # Set, Definition view-structs decoded from entities
  resolve.rs        # effective-set closure, def lookup
  validate.rs       # constraint + cardinality + level checks
  query.rs          # read/traversal API + ChangeSink hang-point
  predicate.rs      # computed >,<,+ over literals
  view.rs           # schema/data/by-type/neighborhood projections
```
km-core has **no** `processing/`. The edge crates each expose a single `src/lib.rs` (plus `hnsw.rs` in
km-edge-enrich-embed) implementing the swap-point traits against km-core's `Graph`/`ChangeSink`.

## Dependencies (Cargo, as built)
| Crate dep | Use | Where / gate |
|-----------|-----|--------------|
| `serde` (derive) | (de)serialize values | all crates, always |
| `bincode` | value encoding for KV + snapshots | km-core, km-edge-enrich-embed |
| `redb` 4.x | embedded ACID KV store | km-core, feature `persist` |
| `hnsw_rs` 0.3 | ANN vector index | km-edge-enrich-embed, feature `hnsw` |
| `serde_json` | extraction IR + LLM payloads | km-edge-enrich-llms, always |
| `reqwest` (blocking, rustls) | Anthropic LLM transport | km-edge-enrich-llms, feature `llm-http` |

Crate edges: km-edge-enrich-embed → km-core; km-edge-enrich-llms → km-core + km-edge-enrich-embed.

Hand-rolled rather than pulling a crate: error enums (manual `Display`/`Error`, no `thiserror`); ids (monotonic `u64` counter, no `uuid`); date literal (`Date{year,month,day}`, no `time`/`chrono`); CLI parsing (`std::env::args` + `match`, no `clap`).
Considered, not used: `fastembed` (ONNX CPU embeddings) — drop-in behind the `Embedder` trait if a learned model is wanted; `candle`/`ort` local inference — out of scope (no GPU target).

## Feature flags (as built)
- **km-core** `persist` — `RedbStore` (redb 4.x) durable backend. Core otherwise has zero features.
- **km-edge-enrich-embed** (Level 1) — base `HashEmbedder` + `VectorIndex` sidecar always on (no extra
  deps); `hnsw` — `HnswIndex` ANN over the sidecar, pulls `hnsw_rs`.
- **km-edge-enrich-llms** (Level 2) — base `Processor` + `MockLlm` + IR always on (pulls `serde_json`);
  `llm-http` — real `AnthropicLlm`, pulls `reqwest` (blocking, rustls).
- (dropped) `local-infer` / Level 3 — out of scope; CPU-only target, no on-device model.
The former in-core `embed`/`llm` features are gone — those layers are now whole crates.

## Layering rule
The edge crates depend on km-core; **nothing in km-core depends on an edge** — enforced by the crate
boundary (km-core builds with no features and has no edge in its dependency tree), not just module
convention. Embeddings are keyed by `Id` in a parallel sidecar, never in `entities`.
