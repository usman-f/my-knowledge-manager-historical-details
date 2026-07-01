# Rust Implementation Plan

Implementation plan for the knowledge-manager backend. Derived from `../design`. Local-first, Rust, Level 0 store first; processing lives in separate edge crates over the pure core (`design/decisions.md`), never inside it.

> **Relationship to `../roadmap/`:** this folder is the *completed* build plan for the core engine (design→code; all phases done, code exists, tests pass). `../roadmap/` is the *forward* co-evolution track — new edge/feature design driven by the example apps (`user-applications/`, now each in its own repo), mostly unbuilt. This folder still serves as the living spec/invariants for what's built; `../roadmap/status.md` reconciles the two.

## Status
All planned work complete. CPU-only, local-first; no GPU / local-inference path (out of scope by decision). 44 tests pass via `cargo test --workspace --all-features`: km-core 26 + `persist` 3; km-edge-enrich-embed 7 + `hnsw` 3; km-edge-enrich-llms 5. km-core also builds with no features. Clippy clean on the default build and every feature combination.
- Phase 3 (Level 1 embeddings → crate `km-edge-enrich-embed`): `ChangeSink` seam in `Graph` (km-core), `Embedder` trait, `VectorIndex` sidecar keyed by id, hybrid (vector ∩ graph) retrieval. Deterministic dependency-free `HashEmbedder` (signed feature hashing) for offline tests; a learned CPU model (`fastembed`, ONNX) is a drop-in `Embedder`.
- Phase 4 (Level 2 LLM processor → crate `km-edge-enrich-llms`): `LlmClient` trait; `Processor` does extraction-driven population with embedding-based entity-resolution/dedup, and GraphRAG (vector entry points → graph traversal → grounded context → answer). Deterministic `MockLlm` for offline tests; real `AnthropicLlm` (blocking reqwest) behind `llm-http` (needs network + `ANTHROPIC_API_KEY`).
- Durable storage (km-core `persist`): `RedbStore` — embedded ACID backend (redb 4.x), CPU-only, single file, same `Store` trait. Entity table is the source of truth; in-memory indexes rebuilt on open; id counter persisted. `MemStore` (+ bincode snapshot) remains for tests/small N.
- Scale retrieval (km-edge-enrich-embed `hnsw`): `HnswIndex` — approximate NN (hnsw_rs, CPU), built from the authoritative `VectorIndex` sidecar; matches brute-force top-k on the example. Rebuild-on-drift sidesteps incremental HNSW deletes.
- Remaining optional swap-ins (behind existing traits, no core change): `fastembed` real embeddings; a generic OpenAI-compatible `LlmClient`. On-device local inference intentionally excluded (GPU-less target).

## Files
- `00-overview.md` — crate/module layout, dependency choices, build profile.
- `01-core-model.md` — Rust types for entities, relationships, literals, ids.
- `02-storage.md` — store trait, in-memory + persistent backends, indexes.
- `03-bootstrap.md` — meta set, primitive literal types, seed data.
- `04-resolution-validation.md` — effective-set closure, definition lookup, validation.
- `05-query-views.md` — read/write/traversal API, computed predicates, views.
- `06-processing-hooks.md` — embedding sidecar + LLM seams (Levels 1–3); the code now lives in the `km-edge-enrich-embed`/`km-edge-enrich-llms` crates.
- `07-milestones.md` — phased delivery, acceptance tests per phase.

## Invariants (from design, enforced in code)
- Two building blocks only: entity, relationship. No `property` type.
- Relationship = `(type: DefRef, target: Target, qualifier: Option<Literal>)`; directed owner→target.
- Target = entity id **or** literal. Literal types = Rust primitives (number/string/bool/date).
- Closed on definitions (every relationship references a def), open on values (absence ≠ false).
- Schema is data: sets, definitions, meta set are ordinary entities.
- `meta.set` composes (union, transitive); `defines` vs `contains` kept distinct.
- Views derived, never stored. Embedding vectors live in a sidecar, never as relationships/literals.

## Stack decisions
- Edition 2021, workspace: pure `km-core` lib + thin `km-cli` bin + `km-edge-enrich-embed`/`km-edge-enrich-llms` edge crates.
- Phase 0–1 storage: `redb` (pure-Rust embedded KV, ACID) or `sled`; default `redb`. In-memory `HashMap` backend behind same trait for tests.
- Serialization: `serde` + `bincode` for values, ids as `u64`/`u128`.
- Errors: hand-rolled enums (manual `Display`/`Error`); ids: monotonic `u64` counter.
- Async deferred until processing layer (Levels 1–3) needs `tokio`/`reqwest`.
