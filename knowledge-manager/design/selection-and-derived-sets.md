# Selection layer & derived-vs-materialized categories

Context: `select.rs` (set/link presence-absence algebra) + the consolidator now
modeling IE status/flags as km set membership. This note records the decision
framework and the km-core enhancements that would let more facts become queryable
data cheaply, across applications (not just the SAP/P&ID consolidator).

## Decision framework — model a fact as queryable data vs. keep it struct-local

Model as store data (set membership / value-entity link / edge) when **all**:
- the carrier is already a persisted entity (membership is a marginal write);
- the fact is intrinsic/stable (true of the entity regardless of the query asking), not relative to a transient computation;
- it is part of the domain's *shared vocabulary* — something an expert asks for or browses, or a second consumer reads (other phases, the explorer, embeddings/LLM edges);
- the query result is the entity (or derivable from it) without re-traversing a parallel struct.

Keep struct-local (single-pass Rust) when **any**:
- the fact is contextual/relative (needs a companion link to be meaningful);
- the grain is not a persisted entity and minting one is a lifecycle commitment;
- it is a single-consumer intermediate of one algorithm;
- materializing creates a second source of truth that can drift.

Guiding rule: **store the domain's vocabulary; compute the algorithm's mechanics.**

## Applied to the consolidator's three grains

- **IE / equipment** (status, should-have-sap, control-valve): domain vocabulary, carrier is the persisted `IdentifiedEquipment` hub → modeled as sets. Fit.
- **Member** (disposition, join-basis): carrier (the source record) is persisted, and disposition *is* domain output (delete reasons). But disposition is **relative to the IE's keeper**, and the consumer (`analyze`) needs keeper/loop context it already has in-struct — a flat set would not remove that traversal, only add writes. Right model is an **edge property** on `ie-has-record` (see D), not a set. Left struct-local pending that.
- **Loop `(unit,var,loop)`**: not an entity; controllers/alarms are deliberately excluded from the IE model, so the loop pass rebuilds from records. Making it queryable means minting a `LoopGroup` hub for every loop and attaching all records — a real modeling commitment. Left struct-local.

## Tension with the locked principle

`design/principles.md` / `design/model.md` §Views: "Views are derived, not stored."
The consolidator currently **materializes** category membership (writes `meta.set`
edges in `build_ie`/`apply_sap_gate`). That works but is against grain: the facts are
derivable from the `ie-status` literal already on the IE. Costs of materialization:
snapshot size (10^5 IEs × N sets), index churn on put, a second source of truth to
keep in sync, and a larger `validate_all` surface. The principled path is a *derived*
category whose membership is computed from the underlying literal/link — which is only
ergonomic if km-core makes derived queries cheap. Hence the enhancements below.

## km-core enhancements (prioritized for cross-purpose use)

### A. Intensional (computed) sets / storable Selections — highest leverage
A set whose membership is a predicate (a stored `Selection` or literal/link
condition), evaluated on read. Self-describing (stored as data like everything else),
so saved queries / domain "lenses" are shareable across apps and the explorer.
- Removes materialization writes + drift; aligns with "views derived, not stored."
- Consolidator would *define* `AtSiteEquipment := ie-status == at-site` once instead of writing membership per IE — and `analyze`'s `Selection` composes over those.
- Cost: a predicate representation in the store + evaluation hook (`members_matching`, or fold into `members_of_set` when a set carries a predicate). Moderate.

### B. Value index `(def, literal) → owners` — broad utility
Today only promoted-to-entity values are reverse-queryable; querying an un-promoted
literal is an O(N) scan. A secondary index keyed by `(def, literal)` makes literal
predicates O(1).
- Every application wants "find entities where field = value" without first promoting.
- Makes A cheap for literal-based predicates.
- Cost: one more `HashMap` in `store::Indexes`, maintained in `index`/`unindex`. Low–moderate.

### C. Effective-membership-aware querying — taxonomy support
`members_of_set` is direct-only; `view::by_type` does build-on-aware membership by
full scan (O(entities)). A reverse index of transitive (build-on) membership, or
memoized `effective_sets`, makes hierarchical category queries cheap.
- Needed the moment categories form a `build-on` taxonomy.
- Cost: incremental maintenance is fiddly (build-on edges mutate membership). Medium.

### D. Reified relationships / edge properties — unlocks the member grain
Let an edge carry its own facts and be indexable: the record→IE edge holding
`{db, disposition, join-basis}`. Today `Relationship.qualifier` is a single optional
literal (already used for `db`), so a second edge fact has nowhere to go.
- Unlocks member-grain querying cleanly; a known KG pattern (RDF-star / property-graph edge props).
- Largest blast radius: `model.rs`, `validate`, persistence, every consumer.
- Defer until a concrete second consumer needs edge-level facts queryable; the single qualifier is the minimal current form.

## Single-source-of-truth refinement

"Views are derived, not stored" conflates two claims with different force:

- (i) **Don't persist what's cheaply recomputable** — an efficiency call, negotiable,
  already relaxed by "cache hot ones". Owning the repo lets you choose to materialize.
- (ii) **Single source of truth** — a correctness property, not repealable by editing
  the doc. Owning the repo does *not* let you keep two independently-written truths.

Three end-states for a fact + its category view:
- **derived-primary** (CHOSEN): canonical = the literal/link; the category set is
  intensional (computed). One truth; the view is a read.
- **materialized-primary**: canonical = the set membership; the literal is a cache.
  Acceptable if one is unambiguously derived from the other.
- **both-independent**: two truths written by different code paths — the only bad one,
  and what the pre-refactor consolidator did. Evidence: the persisted `ie-status`
  literal was **write-only** (`equipment.rs` wrote it; every reader used the in-memory
  `ie.status` struct field), while membership was written separately in
  `build_ie`/`apply_sap_gate`. A latent drift trap for the Phase B/C snapshot consumer.

Decision: **derived-primary**. Canonical fact = the literal/link on the IE; the
category sets are intensional. Membership cannot drift because it is never written.

Generalize, don't flip: km-core supports **both** extensional and intensional sets as a
per-set choice (A adds intensional). A general substrate should mandate neither; the
*application* picks per set.

## Recommendation

Order **B → A → C**, D independent — all now implemented in km-core:
- **B** value index `(def, literal) → owners` (`store`): O(result) literal predicates.
- **A** intensional sets (`derive.rs`): predicate stored as `MembershipCondition`
  entities; `resolve_members` evaluates on read; `select`/`Selection` resolve through it.
- **C** `effective_members` (`select.rs`): build-on-aware, intensional-aware membership.
- **D** edge reification (`reify.rs`): edge-level facts as a stand-in entity; deferred
  as a *mandate* (member/loop grains stay struct-based) — it is the generic capability,
  used when a second consumer needs edge facts queryable.

A realizes derived-primary: the consolidator defines its five IE category sets as
computed sets and drops the materialized `meta.set` category writes; the `ie-status`
and `should-have-sap`/`is-control-valve` literals are the single source of truth.
