# Implementation Review

Scope: correctness, clarity, maintainability, efficiency of `crates/km-core` + `km-cli`. No scope creep — findings are cleanups within the current design. Severity: **fix** (should change) / **consider** (judgment call) / **ok** (noted, no action).

## Overall
Faithful to the design: two building blocks, schema-as-data, closed-defs/open-values, derived views, sidecar embeddings. Module boundaries clean; `Target`/`Literal`/`TargetType` exhaustively matched; invariants tested. Findings below are small.

## Dead code (fix)
- `resolve::def_ids_for` — unused since `validate_entity` was refactored to build `available` from its local `defs`. Remove.
- `store::def(id) -> DefRef` free helper — no callers (tests/CLI/lib). Remove, or use it where `DefRef::new(...)` is written by hand.
- `StoreError::NotFound` — never constructed, only matched. Either drop the variant or return it from a fallible accessor; today `get` returns `Option`, so it is unreachable.

## Correctness (fix / consider)
- **`Processor::populate` relationship cardinality (consider).** Population always *appends* relationships (`push_rel`). Re-running on an entity that already has a `one`-cardinality rel (e.g. `measured-variable`) yields a second edge → `CardinalityError` under `validate`. Acceptable for first-ingest; for idempotent re-ingest, dedup/replace `one` rels. Document the append semantics on `populate` at minimum.
- **`Processor` ignores qualifier type (ok→consider).** Extraction qualifiers are always `Literal::String`; a def whose qualifier is declared numeric/date would mismatch. Current defs don't exercise it; note the limitation.
- **`extract_json` brace-slicing (consider).** `find('{')..rfind('}')` breaks if the model emits prose containing stray braces around the JSON. Fine for `MockLlm`; for real models prefer a tolerant parser or instruct strict JSON-only (the system prompt already says "ONLY JSON").

## Efficiency (consider)
- **`resolve_entity` is O(N · embed) per candidate, O(N²·embed) per document.** It re-embeds every stored entity's `name` on each call. Note the sidecar stores *render* vectors (name + sets + rels), not name-only vectors, so it can't be reused directly for name-based dedup without changing semantics. A clean fix needs a separate name-embedding cache (new state) — deferred with the rest of scale dedup; documented on `populate`. (The `zip` itself is safe: both vectors come from the same `index.embedder`, so dims always match — earlier "length guard" note retracted.)
- **`schema`/`view`/`resolve` call `is_set`/`is_def`/`def_view` repeatedly**, each doing `rels_by_def` scans. For the example (N≈65) irrelevant; at scale these recompute the same `DefView`s. A per-query `DefView` cache (or memoizing `effective_sets`) is the documented hot-path optimization (07-milestones "Cross-cutting"). Leave until the N=100k bench says so.
- **`by_type` scans all ids and computes `effective_sets` for each.** `members_of_set` + reverse build-on traversal would be cheaper, but `effective_sets` is the simplest correct version. Fine at current scale.

## Maintainability (consider)
- **`Store::members_of_set` is unused outside its impls.** It’s defensible as public trait surface, but `set_members` index exists only to back it. Either use it in `by_type`/`resolve` (which would also address the efficiency note) or drop it + the `set_members` index to remove an unused code path.
- **Constraint encoding is stringly-typed** (`range:a..b`, `allowed:…`, `set:Name`) parsed in `schema::parse_constraint`. Matches the design’s "constraint as data" and ships Level 0; a structured form is the noted Phase-2 follow-up. OK.
- **Two persistence formats** (`MemStore` bincode snapshot vs `RedbStore`). Intentional (tests/small-N vs durable). Keep, but the snapshot `save`/`load` could be expressed as "dump to/from `RedbStore`" later to avoid two serializers.
- **`MemStore::rebuild_indexes` is public but only used internally** (load path). Harmless; could be `pub(crate)`.

## Good (ok)
- redb backend keeps the entity table as sole source of truth, indexes rebuilt on open — matches the "indexes are regenerable cache" invariant, no redundant tables.
- `effective_sets` and `closure` are cycle-safe (visited set) — mis-authored build-on cycles terminate.
- Embedding sidecar is keyed by id and never written into `entities` — verified by `embeddings_do_not_touch_the_core_store`.
- Bootstrap fixed-point handled by skipping `is_bootstrap` ids in validation — avoids regress without special-casing reads.
- `f64` ordering via `partial_cmp` with tie-break by id keeps vector search deterministic.

## Applied
1. Removed the three dead items (`def_ids_for`, `store::def`, `StoreError::NotFound`) + now-unused `DefRef` import in `store/mod.rs`.
2. Documented `populate`'s append semantics and O(N) dedup cost on the method.

## Second pass (2026-05-31)

Independent re-review of the full crate. Verified: `cargo clippy` clean on every feature combo (default, embed, hnsw, llm, llm-http, persist); all tests green.

### Fixed
- **`Graph::closure` re-emitted `start` on a cycle.** `start` was never marked `seen`, so an edge back to it (e.g. `A→B→A`) pushed `start` and added it to `order`, contradicting the documented "excludes start". Rewrote: pre-seed `seen` with `start`, push each node once on discovery (drops the `first`-flag bookkeeping and the redundant second dedup). Acyclic output order unchanged. Regression test `closure_excludes_start_through_cycle` added (example test count 16→17).

### Corrections to first pass
- The "is_set/is_def/def_view do `rels_by_def` scans" efficiency note overstates cost: `rels_by_def` is an O(1) `by_def` HashMap lookup (+ small clone), not a scan. Real hot-path quadratic is only `resolve_entity`'s per-call re-embedding (already noted) and `validate_entity`'s O(rels·defs) cardinality count (acceptable at current N).

### New observations (consider, no action)
- **`validate_entity` cardinality loop** recomputes `e.rels.iter().filter().count()` per def — O(rels·defs). A single pass building a per-def count map removes it; deferred behind the N=100k bench with the other hot-path opts.
- **GraphRAG hop expansion pulls schema entities into context.** `entity_neighbors` follows `meta.set` targets, so a set entity can enter `visited` and render `X is-a Set` / build-on lines as context noise. Entry points are already filtered to non-set/def; applying the same filter during expansion would tighten context. Behavior-affecting, so left for a deliberate call.

### Verified correct (no action)
- `effective_sets`/`closure`/`resolve` cycle-safe (visited sets).
- Set/def entities validated only against the universal meta defs; no meta def is `required`, so schema nodes raise no spurious `MissingRequired`.
- `meta.target` (level=schema) forces a def's allowed-set target to be a set; `OutOfSet` checks `effective_sets` membership — both exercised by the negative-path tests.
- redb `next_id` persisted on each `put` (post-alloc `peek`); ids never reused across reopen (`redb_id_counter_persists`).
- Index `unindex` removes the whole `(owner, def)` bucket and retains name/reverse/set_members by id — correct for multi-rel and shared-name entities.

## Third pass (2026-06-10)

Full-repo re-review. `cargo clippy --all-targets --all-features` clean; all tests green (example suite 25→26).

### Fixed
- **Set-name lookups could be shadowed by a data entity sharing the name.** `resolve_def`, `parse_constraint`'s `set:` branch, `populate`'s set assignment, and the CLI `by-type` all took the *first* `by_name` match, set or not — a data entity named like a set silently broke def resolution / constraint decoding / set assignment. Added `schema::set_by_name` (first match that `is_set`) and routed all four through it. Regression test `resolve_def_skips_data_entity_sharing_set_name`.
- **`validate_entity` double def lookup.** The `available: HashSet<Id>` membership check duplicated the `defs.iter().find(..)` that followed (whose `else { continue }` was unreachable). Collapsed to a single `find` with let-else; HashSet and import dropped.
- **`MemStore::rebuild_indexes` cloned every entity** just to appease borrows; disjoint field borrows (`self.idx` mut, `self.entities` read) make the clone unnecessary. Removed.
- **Dot product duplicated** inline in `llm::resolve_entity`; now reuses `embed::dot` (`pub(crate)`).
- Added a root `README.md` (repo layout, build/test).

## Left as-is (justified)
- `resolve_entity` O(N²): correct as written; reuse-stored-vectors would change dedup semantics (render vs name vectors), so it needs a dedicated cache — deferred with scale dedup.
- `DefView`/`is_set` recomputation, `by_type` full scan: simplest-correct; optimize behind the N=100k bench (07-milestones cross-cutting).
- `members_of_set` trait method, stringly-typed constraints, dual persistence formats: intentional API/design surface, kept.
