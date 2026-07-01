# Step 15 — Embeddings & vector search *(advanced)*

Files: `km-edge-enrich-embed/src/lib.rs`, `hnsw.rs` (feature `hnsw`). Level 1 of the
processing layer — a separate edge crate over km-core, not an in-core module (the old
`src/processing/embed.rs` behind feature `embed` is gone; moved out 2026-06-24).

## Understand

**Why is the vector a *derived, regenerable* artifact and never a relationship?**
- `render_entity` flattens an entity to text (name + effective-set names + each
  relationship's def name, target label, qualifier); `Embedder::embed` turns that into a
  fixed-length `Vec<f32>`; it's stored in a `VectorIndex` sidecar keyed by entity id —
  a `BTreeMap<u64, Vec<f32>>`, persisted *separately* (`save`/`load`). Deleting the
  sidecar loses nothing durable; it's rebuilt by re-embedding.
- Keeping it out of the core store preserves the store's exactness: the vector is
  model-specific and lossy, the antithesis of a symbolic fact. The edge crate depends on
  km-core, never the reverse — now enforced by the **crate dependency direction**
  (km-core has no dependency on any `km-edge-*` crate), not just a module convention.

**How does `EmbeddingIndex` stay fresh?**
- `impl<E: Embedder> ChangeSink for EmbeddingIndex<E>`: `on_put` re-embeds the touched
  entity, `on_delete` removes its vector. Wire it into a `Graph` via `add_sink` and the
  store's change notifications keep vectors in lockstep — the "hang-point" the design
  asked Level 0 to leave open.

**Why can a real model be swapped in by implementing `Embedder` alone?**
- Generic code targets the `Embedder` trait (`dim`/`tag`/`embed`), not a concrete model.
  The built-in `HashEmbedder` (deterministic signed feature-hashing — offline-testable)
  and a future ONNX BGE/MiniLM model are interchangeable; nothing else changes.
- `tag` guards comparability: all vectors in an index must come from the same model
  (same `tag`/`dim`); switching models means re-embedding everything.

## Rust

- Static dispatch: `EmbeddingIndex<E: Embedder>` is monomorphized per embedder (zero
  runtime cost) — the counterpart to `query.rs`'s `dyn ChangeSink` (dynamic dispatch).
  Two kinds of polymorphism, chosen per use.
- `impl Iterator<Item = String> + '_` (in `tokenize`): the `+ '_` ties the iterator's
  lifetime to the borrowed `text` it slices into.
- `&mut [f32]` mutable slice (`normalize`); `iter_mut()` yields `&mut`, `*x /= norm`
  writes through it.
- `zip` pairs two iterators for the dot product; `sum::<f32>()` turbofish names the
  accumulator type.
- Closures as generic `Fn` params: `hybrid<F: Fn(Id) -> bool>(.., filter: F)`; `.take(k)`
  lazily stops after k.

## Hybrid retrieval

`hybrid(query, k, filter)` ranks *all* candidates by vector similarity, keeps those the
graph `filter` admits, returns top-k — vector meaning ∩ exact graph constraints
(`has_relation_to` builds such filters). This is the core of GraphRAG (Step 16).

`HnswIndex` (feature `hnsw`) is *built from* the authoritative `VectorIndex` for fast
approximate NN at scale ("brute force at small N; HNSW at scale"), rebuilt when the
sidecar drifts — sidestepping HNSW's awkward incremental deletes.
