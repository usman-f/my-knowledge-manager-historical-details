# Approach Review

Higher-level review of intent and approach (2026-05-31). Code-level cleanups are in `review-notes.md` and not repeated here. Severity: **gap** (design promise not met) / **consider** (judgment call) / **strong** (working well).

## Intent (as read)
Local-first Rust knowledge manager. One uniform entity/relationship structure where schema is itself data (meta-circular, RDFS/SHACL-shaped). Literals as a typed floor; promotion to entities only when knowledge attaches. Sets = types via composition. Closed on definitions, open on values. Fuzzy/learned capability (embeddings, LLM) as decoupled layers above a pure symbolic store. CPU-only.

## Strong (keep)
- Design→implementation traceability (doc-referencing module comments) is unusually disciplined; the meta-circular bootstrap is clean and actually self-describing.
- Layer boundary enforced structurally: the edge crates (`km-edge-enrich-embed`/`km-edge-enrich-llms`) → core, never reverse; vectors in an id-keyed sidecar, never in `entities`. Verified by test.
- Exhaustive matches on `Target`/`Literal`/`TargetType`; cycle-safe `effective_sets`/`closure`; deterministic, dependency-free defaults (`HashEmbedder`, `MockLlm`) make the whole stack testable offline.

## Approach-level considerations

### 1. No edit/retract path — wrong primitive for a "manager" (gap) — FIXED
- Was: `Graph` write API was `add_entity` / `relate` (append-only) / `delete` (whole entity). No remove-relationship, no replace-one-cardinality, no retraction. (`05-query-views.md` had planned a `remove_rel` that was never built.)
- Fixed: added `Graph::unrelate(owner, def, target?)` (retract one edge or all of a def), `set_one(owner, def, target, q)` (replace-or-insert for `one`-cardinality), and `unassign_set` sugar. All route through `put` so indexes + `ChangeSink` stay consistent; `unrelate` is a no-op (no notify) when nothing matched. Tests cover replace-in-place, single/bulk retract, reverse-index update, membership/build-on drop, and sink notification (incl. no-op suppression).
- Also fixed: whole-entity `delete` no longer leaves dangling references. Both backends (`MemStore`, `RedbStore`) now strip inbound links from their owners before removing the entity (shared `inbound_owners` helper over the reverse index), so the link invariant — every link's two endpoints both exist, and both track it (outgoing rel on the owner, reverse edge on the target) — holds after delete. `store::dangling_targets` audits it; `Graph::delete` refreshes the stripped owners' derived artifacts (embeddings) too. The reverse index is the "both endpoints track the link" bookkeeping; relationships remain directed and stored once (no materialized back-edges), per the design.

### 2. Schema is unvalidated (gap)
- Data is validated against schema; nothing validates the schema. Set/def entities are checked only against the universal meta defs (`validate_entity`), which carry no constraints and nothing `required` — so malformed schema passes silently.
- Failure modes that slip through: a def with `target-type=entity-set` *and* a numeric `range:` constraint; `target-level` on a literal target; a `meta.defines` pointing at a non-def; a def with no `target-type`.
- Compounding: `parse_constraint` returns `None` on any parse error, so a typo'd `range:0..10x` silently becomes *no constraint* rather than an error.
- Option: a schema-lint pass (run over `is_set`/`is_def` entities) asserting def well-formedness; make constraint parse failure a `Violation`, not a silent drop.

### 3. Literal values are unindexed — range/value queries can't scale (gap vs. promise)
- `design/decisions.md` justifies keeping literals partly for "indexing/compare/sort/range." The reverse index (`Indexes::reverse`) only records `Target::Entity`; literal targets are not indexed at all.
- `predicate.rs` computes `>`/`<`/range but there is no structure to answer "instruments with range-high > 50" without scanning every entity.
- Option: an optional `(def → sorted literal values → owners)` sidecar index, same regenerable-cache stance as the existing indexes. Defer building, but the design's range-query claim is currently unbacked.

### 4. Qualified-name identity is a convention over id identity, with sharp edges (consider)
- Design: "the qualified `Set.name` *is* the identity." Implementation: identity is `Id`; `resolve_def` reconstructs it via `by_name(set).first()` + `split_once('.')`.
- Edges: `by_name` returns first match, so two sets sharing a name silently alias; `split_once('.')` breaks for any set/def name containing `.`; resolution is O(scan) per lookup.
- Today's data avoids all three. Option: forbid `.` in set/def names at write time, or carry the qualified name as a stored, uniqueness-checked field rather than recomputing it.

### 5. Durable backend isn't wired into the actual tool (consider)
- `RedbStore` (ACID, "the local-first durable backend") exists and is tested, but `km-cli` reads/writes only the `MemStore` bincode snapshot. The user-facing path is the non-durable one; the durable one has no entry point.
- Option: a `--store redb|snapshot` flag (or make redb the CLI default), so the durable story is reachable, not just unit-tested.

### 6. Depth over breadth — processing layers ahead of a corpus (consider)
- Built vertically through Level 2 (LLM/GraphRAG) + HNSW + redb, but the only dataset is the one hand-coded ISA loop (~65 entities). HNSW and `llm-http` are scale/production concerns with no corpus to exercise them.
- Milestones say "bench `effective_sets` + vector search at N=100k before optimizing (HNSW threshold)" — no bench or criterion dep exists, so HNSW shipped ahead of the evidence its own plan asked for.
- Option: before more layers, invest in (a) a real ingestion path + a non-toy dataset, (b) the edit/query ergonomics (items 1, 3), (c) the deferred bench — then let measured N decide HNSW/caching.

### 7. Prior-art positioning is implicit (consider)
- The model is effectively RDF (triples + reification via the qualifier) with an RDFS/SHACL-style schema-as-data layer; docs note the RDF analogy in passing but never state why build-from-scratch over an embedded triple store / `oxigraph` / a SHACL validator.
- For an exploratory local-first project, from-scratch is defensible (full control, no deps, learning value). Worth one explicit `decisions.md` entry so the trade-off is a recorded choice, not an omission.

## Smaller code observations (new, not in review-notes)
- `IdGen` uses `AtomicU64` but every `alloc_id` path is `&mut self` — the atomic buys nothing; a plain `u64` is equivalent and clearer.
- `defs_for` dedups by qualified name "first wins"; on a genuine name clash across composed sets this hides the loser silently rather than flagging an ambiguity.
- No property/roundtrip tests for the foundational invariants (put/delete↔index consistency, serialize roundtrip, constraint-parser fuzz). For a data-model core these would harden more than additional example-shaped unit tests.

## Suggested sequence
1. Edit/retract API (1) + make constraint-parse failure a violation (2). Small, high-leverage, unblock real editing.
2. Schema-lint pass (2).
3. Real ingestion + dataset; wire redb into CLI (5, 6a).
4. Bench (6); literal index only if the bench says so (3).
