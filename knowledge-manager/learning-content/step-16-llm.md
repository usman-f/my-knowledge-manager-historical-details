# Step 16 — LLM processor (extraction + GraphRAG) *(advanced)*

File: `km-edge-enrich-llms/src/lib.rs` (`llm-http` feature for the real client). Level 2
— a separate edge crate over km-core + km-edge-enrich-embed (the old `src/processing/llm.rs`
behind feature `llm` is gone; moved out 2026-06-24).

Scope note: the crate's *forward* role is the **enrich** stage (gap-fill / mistake-find
→ human-gated claims, never direct fact writes). The two flows below — `populate`
(document extraction, a *load* path) and `answer`/`graphrag_context` (a *use* path) — are
built and instructive but sit outside that scope; treat them as example/legacy
(`roadmap/status.md`, `design/processing-layer.md` §Level 2).

## Understand

**Which part is "fuzzy" vs "exact and testable"?**
- Fuzzy: the single model call behind `LlmClient::complete` (extraction prompt;
  answer prompt). Swappable: `MockLlm` (canned, offline) vs `AnthropicLlm` (real, behind
  `llm-http`) — the dependency-injection pattern, one trait two impls.
- Exact / deterministic / testable: everything else — JSON slicing (`extract_json`) +
  parse (`serde_json` into `Extraction`), entity resolution/dedup (`resolve_entity`),
  context assembly (`graphrag_context`), and the writes (`push_rel`). `MockLlm` lets the
  whole orchestration be tested without a network.

**Trace `populate` (write).**
1. Build user prompt: `schema_digest(store)` + the document.
2. `llm.complete(EXTRACT_SYSTEM, ..)?` → raw text; `extract_json` tolerates prose →
   `serde_json::from_str` → `Extraction { entities, relationships }` (`map_err` →
   `LlmError::Decode`).
3. Entities first: for each, `resolve_entity` (exact data-name match, else nearest data
   entity by name embedding above `merge_threshold`). If found → reuse; else create,
   assign its `sets` via `meta.set`, `index.update`. Record `key → id` in `keymap`.
4. Relationships: resolve `owner`, `def` (`resolve_def("Set.name")`), and target
   (entity via `resolve_ref`, or a literal number/string/bool); `push_rel` +
   `index.update`.
- Relationships are **appended**, not reconciled — re-ingesting a `one`-cardinality fact
  adds a second edge that `validate` then flags (idempotent re-ingest is future work).

**Trace `graphrag_context` (read).**
- `index.hybrid(question, top_k, |id| data-entity)` → vector entry points among data
  entities.
- BFS expand `hops` along `entity_neighbors` (outgoing entity targets + incoming edge
  owners), deduped via `visited`.
- For each visited entity, `entity_facts` renders domain facts (`Owner [q] def -> target`,
  `meta.set` as `is-a`; meta plumbing skipped), deduped, joined. `answer` wraps this as
  grounded context for a final `llm.complete(ANSWER_SYSTEM, ..)`.

**Why embed the candidate before creating an entity (dedup)?**
- Entity resolution: exact-name matching misses near-duplicates ("Unit 02" vs "unit-02"),
  the classic KG failure mode. Embedding the candidate name and vector-searching existing
  entities catches cosine-similar matches ≥ `merge_threshold` and merges into the
  existing id instead of minting a duplicate.

## Rust

- Idiomatic error type: `enum LlmError` + `Display` + `impl std::error::Error {}` (empty
  marker body) so it composes with `?` and `Box<dyn Error>`.
- serde `#[derive(Deserialize)]` + `#[serde(default)]` — missing JSON keys fall back to
  `Default` (empty Vec / `None`), so the model may omit `sets`/`set` without a parse
  failure.
- `?` propagates `Err`; `.map_err(...)` adapts one error type to another before `?`.
- Feature-gated transports = dependency injection: `MockLlm` always compiled,
  `AnthropicLlm` + its `reqwest` dependency only with `llm-http`.

## The shape of it

Embeddings find the door (entry points by meaning), the graph walks the rooms (exact
connected facts), the LLM explains what's inside (grounded answer). The store stays pure;
only `complete` is fuzzy.
