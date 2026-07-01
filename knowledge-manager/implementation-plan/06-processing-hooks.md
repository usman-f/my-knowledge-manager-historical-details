# Processing Hooks — Levels 1–2 (implemented) + scale

> **Moved 2026-06-24:** this layer now lives in the edge crates `km-edge-enrich-embed` (L1) and
> `km-edge-enrich-llms` (L2), not `km-core/src/processing/` (`design/decisions.md`). The seams and
> symbols are unchanged; paths/features below are repointed to the crates. The `ChangeSink` seam stays
> in km-core (`query.rs`); the former in-core `embed`/`llm` features are gone (each layer is now a
> whole crate), while `hnsw` and `llm-http` remain optional features on their crates.

Implemented: Level 1 embeddings (crate `km-edge-enrich-embed`), Level 2 LLM processor (crate `km-edge-enrich-llms`, `llm-http`), and an HNSW ANN index (`hnsw`, `km-edge-enrich-embed/src/hnsw.rs`) built from the `VectorIndex` sidecar for retrieval at scale. Level 3 local inference is out of scope (CPU-only target).

Store stays pure. Processing attaches as swappable layers above the `Store`/`Graph` API. Build the Level-0 core with the seams below so Levels 1–3 wire in without touching the symbolic model.

## Seam contract (built in Phase 0, feature-gated impls later)
- `Graph` read/write/traversal API (05) is the only surface processors use.
- A place to **hang one embedding per entity** — a sidecar keyed by `Id`, never a relationship/literal in `entities`.
- `put`/`delete` emit change events so derived artifacts (vectors, caches) can stay fresh.
```rust
pub trait ChangeSink { fn on_put(&mut self, store: &dyn Store, id: Id); fn on_delete(&mut self, id: Id); }
// Graph holds Vec<Box<dyn ChangeSink>>; core ignores them, processors subscribe.
```

## Level 1 — embeddings + vector search (crate `km-edge-enrich-embed`)  [implemented]
Built: `Embedder` trait, deterministic `HashEmbedder`, `VectorIndex` sidecar (brute-force cosine, bincode-persisted, dim/tag guard), `EmbeddingIndex` (`ChangeSink`), `render_entity`, `hybrid()`. ANN at scale via `HnswIndex` (feature `hnsw`, `km-edge-enrich-embed/src/hnsw.rs`). Optional: a learned `fastembed` (CPU/ONNX) `Embedder` drop-in.

```rust
pub trait Embedder { fn dim(&self) -> usize; fn embed(&self, text: &str) -> Vec<f32>; }
pub struct VectorIndex { hnsw: Hnsw, dim: usize, model_tag: String }  // sidecar file, by Id
```
- One vector per entity: render entity → text (`name + effective sets + key rels`), embed, store in sidecar. Recompute on `on_put`.
- `fastembed` (BGE/MiniLM ONNX, CPU) for compute; `hnsw_rs` (disk-persisted) for index. Brute-force fallback at small N.
- Normalize vectors; cosine/dot. Size ≈ N·dim·4 B. `model_tag` guards comparability — switching models ⇒ re-embed all.
- **Hybrid retrieval**: `vector_search(query) ∩ graph_filter` (e.g. nearest "flow sensor" ∩ `located-in = Unit-02`). The query side composes `VectorIndex` results with `Graph::neighbors`/`by-type` — no new storage.

## Level 2 — LLM processor (crate `km-edge-enrich-llms`)  [implemented]
Built: `LlmClient` trait; `Processor::{populate, graphrag_context, answer}`; extraction JSON IR; embedding-based entity resolution/dedup; deterministic `MockLlm`. Real `AnthropicLlm` (blocking reqwest) behind `llm-http`.

```rust
pub trait Processor {
  fn populate(&self, doc: &str, g: &mut Graph) -> Vec<Id>;   // extract entities/rels, with dedup
  fn answer(&self, q: &str, g: &Graph, v: &VectorIndex) -> String; // GraphRAG
}
```
- **Population**: extract candidate entities/rels; before `add_entity`, embed candidate + vector-search for near-duplicates ⇒ entity resolution/dedup (catches what exact-name matching misses).
- **Use (GraphRAG)**: embed question → vector search entry points → `Graph::closure`/`neighbors` for connected facts → hand subgraph to LLM as grounded context.
- Transport: `reqwest`/`tokio` to an LLM API (simplest, most capable).

## Level 3 — local inference  (out of scope)
- Dropped by decision: target is a GPU-less machine. CPU embeddings (`fastembed`/ONNX) + a remote `LlmClient` cover the need. The `Embedder`/`LlmClient` traits remain the swap points if revisited.
- Not Rust's job in any case: training. Only ever inference over frozen pre-trained weights; "learning" stays offline (swap model) or recorded as store data.

## Boundary enforcement
- The edge crates depend on km-core; km-core never depends on them and compiles with no features — enforced by the crate boundary, not just module convention.
- Vectors/caches are regenerable, model-specific artifacts kept out of `entities`/`reverse` tables — deleting the sidecar loses nothing durable.
