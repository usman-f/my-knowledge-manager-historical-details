# Implementation plan: value index, intensional sets, effective membership, edge reification + consolidator switch to derived-primary

Self-contained spec for a fresh session. Two repos, checked out as siblings:
- `my-knowledge-manager` — km-core (generic). Crate root: `knowledge-manager/crates/km-core/`.
- `my-km-data-consolidator` — app. Crate: `crates/data-consolidator/`. Depends on km-core by path.
- Both develop on branch `claude/festive-cerf-uw306v`.

Build/test: `cargo test` in each crate. clippy must stay clean (`cargo clippy --all-targets`).

## Goal

Switch the consolidator's IE category facts (status, should-have-sap, control-valve)
to **derived-primary**: the canonical fact is a literal on the IE; category sets are
**intensional** (membership computed from a stored predicate), not materialized
`meta.set` edges. Build the generic km-core capabilities that make this cheap:
B value index, A intensional sets, C effective membership, D edge reification.

Chosen because: the set definition stays visible at the query site (`analyze` reads
`AtSiteEquipment ∩ ShouldHaveSap`, each defined as one predicate) instead of
membership being written by logic elsewhere; and it restores single-source-of-truth
(the `ie-status` literal is currently write-only — a latent drift trap).

## Current state (already committed on the branch — this plan REPLACES parts of it)

`crates/data-consolidator/src/equipment.rs` currently MATERIALIZES the category sets:
- `IeSchema` has fields `confirmed_set, sap_only_set, at_site_set, should_have_sap_set, control_valve_set`.
- `seed_ie_schema` creates them as plain `meta.kind=set` entities via a `cat()` closure.
- `build_ie` writes `meta.set` membership to the status set and (if CV) the control-valve set.
- `apply_sap_gate` writes `should_have_sap` membership via an `add_to_set` helper.
- `Consolidation.cats: IeCats { confirmed, sap_only, at_site, should_have_sap, control_valve }`.
- `analyze.rs` selects add-eligible IEs: `Selection::new().in_set(con.cats.at_site).in_set(con.cats.should_have_sap).run(store)`.
- Test `crates/data-consolidator/tests/equipment.rs::category_sets_mirror_ie_facts_and_select` asserts membership via `store.members_of_set(...)`.

The plan keeps `IeCats`, the cats field, and the `analyze` call site, but changes the
sets from extensional/materialized to intensional/derived, makes the literals
canonical, and removes the materialized writes (`add_to_set`, the membership pushes).

---

## Feature B — value index `(def, literal) → owners`

File: `crates/km-core/src/store/mod.rs`. RedbStore (`store/redb.rs`) shares the same
`Indexes`, so it needs no logic beyond the trait method delegating to `self.idx`.

1. `Literal` is NOT `Hash`/`Eq` (`Number(f64)`). Add a private hashable key:
   ```rust
   #[derive(Clone, PartialEq, Eq, Hash)]
   enum ValueKey { Num(u64), Str(String), Bool(bool), Date(i32, u8, u8) }
   fn value_key(l: &Literal) -> ValueKey  // Number(n) -> Num(n.to_bits()); Date -> (y,m,d)
   ```
2. `Indexes` (the `#[derive(Default)] struct Indexes`): add field
   `by_value: HashMap<(Id, ValueKey), Vec<Id>>`.
3. `Indexes::index`: for each rel with `Target::Literal(l)`, push `e.id` to
   `by_value.entry((r.def(), value_key(l)))`. `Indexes::unindex`: `retain(|x| *x != e.id)`
   on that bucket (mirror the existing `set_members`/`reverse` maintenance).
4. `Store` trait: add `fn owners_with_value(&self, def: Id, value: &Literal) -> Vec<Id>;`.
   - MemStore + RedbStore impl: `self.idx.by_value.get(&(def, value_key(value))).cloned().unwrap_or_default()`.
   - `value_key` must be reachable from both impls (same module).
5. Re-export nothing new from `lib.rs` for the trait method (it's a `Store` method).
   Add a `select.rs` free fn `pub fn owners_with_value(store, def, value) -> Vec<Id>`
   that sorts+dedups, for ergonomic callers.

Tests (in `store/mod.rs` `#[cfg(test)]` or `tests/`): put entities with literal rels,
assert `owners_with_value` returns exact owners; number key (incl. equal-after-to_bits),
string, bool; absent value → empty.

---

## Feature A — intensional (computed) sets

New file: `crates/km-core/src/derive.rs`. Add `pub mod derive;` to `lib.rs`.

### Critical validation constraint (discovered in `validate.rs`)

`validate_entity` validates a **set or def entity against ONLY the meta defines**
(`is_set || is_def` → `defs = defines_of(SET_META)`), NOT against its build-on
defines. Therefore you CANNOT hang a non-meta predicate def (e.g. `member-when`)
on the intensional set entity — it would be flagged `UndefinedRelationship`.

Design consequence: **store the predicate as separate condition data-entities that
point AT the set**, not as rels on the set. The intensional set entity stays a
plain `meta.kind=set` (clean). The predicate = the set of condition entities whose
`cond-for-set` targets it (found via the reverse index).

### cond-value typing constraint

`validate` enforces literal type == def target-type. Status value is `String`,
flags are `Bool`. A single typed `cond-value` def can't hold both. Use **per-type
value defs**: `cond-value-str` (string), `cond-value-bool` (bool), `cond-value-num`
(number), `cond-value-date` (date). The condition writes whichever matches the
literal's type; the evaluator reconstructs the `Literal` from whichever is present.
(Consolidator only exercises str + bool.)

### Schema (seeded idempotently by name inside derive.rs)

km-core has no `ensure_*` helpers; add private ones in derive.rs (find by name with
`schema::is_set`/`is_def`, else create — mirror `km-edge-promote::ensure_def`).

- Set `MembershipCondition` defining: `cond-for-set` (entity→set, one), `cond-kind`
  (string, one: `has-value`|`linked-to`|`in-set`), `cond-negate` (bool, one),
  `cond-def` (entity, one — a def or link def), `cond-target` (entity, one),
  `cond-set` (entity→set, one), `cond-value-str|bool|num|date` (typed, one).
  Set level on entity-target defs to `None` (permissive) to avoid `LevelMismatch`
  when targeting def entities.
- A condition entity is a member of `MembershipCondition` (so its rels validate).

### API

```rust
pub enum Cond {
    HasValue { def: Id, value: Literal, negate: bool },
    LinkedTo { def: Id, target: Id, negate: bool },
    InSet    { set: Id, negate: bool },
}
/// Create a `meta.kind=set` entity named `name` and one condition entity per `conds`
/// (each linked to it via `cond-for-set`). Returns the set id.
pub fn define_intensional_set<S: Store>(store: &mut S, name: &str, conds: &[Cond]) -> Id;

/// Members of `set`: if it has any `cond-for-set` conditions → evaluate the predicate;
/// else → `store.members_of_set(set)` (extensional). Cycle-safe.
pub fn resolve_members<S: Store + ?Sized>(store: &S, set: Id) -> Vec<Id>;
```

### Evaluator (`resolve_members`, with a private cycle-guard variant)

1. conditions = incoming `cond-for-set` edges to `set` → owner condition entities.
   If empty → return `store.members_of_set(set)`.
2. Split into positives (negate=false) and negatives (negate=true).
3. Per-condition candidate id-set:
   - `HasValue{def,value}` → `store.owners_with_value(def, &value)` (feature B).
   - `LinkedTo{def,target}` → `select::linked_to(store, def, target)`.
   - `InSet{set}` → `resolve_members(store, set)` recursively (pass the visited guard).
4. candidates = intersection of all positives, or `store.ids()` if no positives.
   Subtract every negative's id-set. Return sorted (reuse `select::sorted`).
5. Cycle guard: `HashSet<Id>` of sets currently being resolved; if `set` already in
   it, return empty (break the cycle).

### Make set-algebra intensional-aware

In `select.rs`, change `members_in_all`, `members_in_any`, `members_with_without`,
and `Selection::run`'s positive/negative set lookups to call
`crate::derive::resolve_members(store, s)` instead of `store.members_of_set(s)`.
(Behavior identical for extensional sets; intensional sets now resolve.) This is what
lets `analyze`'s existing `Selection` call work unchanged over intensional cats.
Mutual module refs select↔derive are fine in Rust.

`lib.rs`: `pub use derive::{define_intensional_set, resolve_members, Cond};`.

Tests: define a `has-value` set, assert `resolve_members` == owners; `Selection`
`in_set` over it; negation; `in-set` referencing another intensional set (recursion);
self-cycle returns empty, not stack-overflow.

---

## Feature C — effective (build-on-aware) membership

`view::by_type` already computes this by scanning all ids + `effective_sets` (O(N)).
Add a queryable primitive that closes over the build-on hierarchy instead.

Add `pub fn effective_members<S: Store + ?Sized>(store: &S, set: Id) -> Vec<Id>` (in
`select.rs`):
- An entity is an effective member of `set` iff `set ∈ effective_sets(entity)` =
  one of its direct sets is `set` or transitively builds-on `set`.
- `SET_META` is universal → return `store.ids()`.
- Else: `want = {set} ∪ {set entities that transitively build-on set}`. Compute by
  fixpoint over set entities (`schema::is_set`): a set `s` joins `want` if
  `store::direct_sets(s)` (its build-on parents) intersects `want`. Union
  `derive::resolve_members(store, x)` for `x ∈ want` (honors intensional sets), sorted.
- `lib.rs`: `pub use select::effective_members;`.

Tests: A builds-on B builds-on C; `effective_members(C)` includes members assigned to
A/B/C; `effective_members(SET_META)` = all; intensional member set honored.

---

## Feature D — edge reification (non-invasive)

New file `crates/km-core/src/reify.rs`; `pub mod reify;`. Uses only existing
primitives (entities+rels) — no change to `model.rs`/persistence. Lets an edge carry
its own facts and be queried (and, via B, found by property value).

### Schema (seeded idempotently)

Set `ReifiedEdge` defining: `rel-subject` (entity, one), `rel-predicate` (entity, one
— the predicate def being reified; level None), `rel-object` (entity, one). Edge-prop
defs are app-specific and authorized by the caller (see below).

### API

```rust
/// Create a reified-edge entity (member of ReifiedEdge) linking subject -predicate-> object,
/// carrying `props` (extra rels = edge facts). `prop_defs` are added to ReifiedEdge.defines
/// (idempotent) so the props validate. Returns the edge id.
pub fn reify<S: Store>(store, subject: Id, predicate: Id, object: Id, prop_defs: &[Id], props: Vec<Relationship>) -> Id;
pub fn edges_from<S: Store + ?Sized>(store, subject: Id) -> Vec<Id>; // via reverse rel-subject
pub fn edges_to<S: Store + ?Sized>(store, object: Id) -> Vec<Id>;    // via reverse rel-object
```

Finding edges by property value uses `store.owners_with_value(prop_def, &value)`
filtered to `ReifiedEdge` members (or just rely on the prop def being edge-only).

`lib.rs`: `pub use reify::{reify, edges_from, edges_to};`.

Tests: reify an edge with a `disposition` string prop; `edges_from(subject)` /
`edges_to(object)` find it; `owners_with_value(disposition_def, "dedup")` finds it;
`validate_all` clean (props authorized).

Note for future (out of scope): native multi-qualifier / edge-property storage on
`Relationship` is the heavier alternative; deferred because it breaks the bincode
snapshot shape and touches every consumer. This helper is the minimal real form.

---

## Consolidator (b) — switch to derived-primary

File `crates/data-consolidator/src/equipment.rs`:

1. `seed_ie_schema`:
   - Add two canonical bool flag defs on the IE: `should-have-sap`, `is-control-valve`
     (bool, one). Authorize both in `IdentifiedEquipment`'s `meta.defines` (alongside
     `std_tag`, `d_loop`, `d_status`, ...). Keep `d_status` ("ie-status") as the
     canonical status literal.
   - REMOVE the `cat()` closure that made extensional sets.
   - DEFINE the five sets intensionally via `km_core::derive::define_intensional_set`:
     - `ConfirmedEquipment`  = `[HasValue{ d_status, Literal::String("confirmed"), false }]`
     - `SapOnlyEquipment`    = `[HasValue{ d_status, "sap-only", false }]`
     - `AtSiteEquipment`     = `[HasValue{ d_status, "at-site", false }]`
     - `ShouldHaveSap`       = `[HasValue{ should_have_sap_def, Literal::Bool(true), false }]`
     - `ControlValveEquipment` = `[HasValue{ is_control_valve_def, Literal::Bool(true), false }]`
       (status strings must equal `IeStatus::label()`: "confirmed"/"sap-only"/"at-site".)
   - Keep `IeSchema` fields + `IeCats` (now holding intensional set ids).
2. `build_ie`:
   - REMOVE the `meta.set` membership pushes for `status_set` and `control_valve_set`.
   - WRITE canonical literals: `Relationship::new(schema.is_control_valve_def, Literal::Bool(is_control_valve))`
     (always, true or false). `d_status` literal already written — keep.
3. `apply_sap_gate`:
   - REMOVE `add_to_set(...)` calls and the `add_to_set` helper.
   - After computing `ie.should_have_sap`, WRITE the canonical flag onto the IE entity
     in the store: load `ie.id`, push `Relationship::new(should_have_sap_def, Literal::Bool(ie.should_have_sap))`, put.
     (Single source of truth: this literal is what `ShouldHaveSap` derives from.)
   - Signature can drop `schema`/store threading only if no longer needed — it still
     needs `store` + the flag def to write the literal.
4. `Consolidation.cats` unchanged in shape; populated from the intensional set ids.

File `crates/data-consolidator/src/analyze.rs`: NO change — the existing
`Selection::new().in_set(con.cats.at_site).in_set(con.cats.should_have_sap).run(store)`
now resolves intensionally (because select calls `resolve_members`). Confirm the
`Selection`/`Id` imports remain.

### Tests to update (consolidator)

- `tests/equipment.rs::category_sets_mirror_ie_facts_and_select`: it currently reads
  membership via `store.members_of_set(c.confirmed)` — intensional sets have NO
  `meta.set` members, so that returns empty. CHANGE to
  `km_core::resolve_members(&store, c.confirmed)` (and same for the others), or to
  `Selection::new().in_set(...).run(&store)`. Keep the assertions (membership must
  still mirror `ie.status` / `ie.should_have_sap` / `ie.is_control_valve`).
- `tests/equipment.rs::graph_validates_clean`: must stay clean — verify the condition
  entities, the two new flag literals, and the intensional set entities all validate
  (conditions via `MembershipCondition.defines`; flags via `IdentifiedEquipment.defines`;
  intensional sets carry only `meta.kind=set`). If a violation appears, it's almost
  certainly a missing authorization (add the def to the right set's `defines`) or a
  level/type mismatch on a `cond-*` def.
- `tests/set1.rs` (4 tests) + the rest of `tests/equipment.rs`: must pass unchanged
  (behavior identical; only the representation of the cats changed).

---

## Design note (a) — fold in single-source-of-truth refinement

Edit `knowledge-manager/design/selection-and-derived-sets.md`. Add a subsection
(dense, factual):

- Split "Views are derived, not stored" into (i) *don't persist what's cheaply
  recomputable* — negotiable, already relaxed ("cache hot ones"); and (ii) *single
  source of truth* — a correctness property, not repealable by editing the doc. Owning
  the repo lets you choose to materialize, not to keep two independently-written truths.
- Three end-states: derived-primary (CHOSEN), materialized-primary, both-independent
  (the only bad one — and what the pre-refactor code did). Concrete evidence: the
  persisted `ie-status` literal was **write-only** (`equipment.rs` wrote it; every
  reader used the in-memory `ie.status` struct field) — a latent drift trap for the
  Phase B/C snapshot consumer.
- Decision: derived-primary; canonical = literal/link, category sets intensional.
- Generalize, don't flip: km-core supports BOTH extensional and intensional sets as a
  per-set choice (this plan adds intensional); a general substrate should mandate
  neither. Update the "Recommendation" so A = realize derived-primary; consolidator
  drops materialized writes.

---

## Validation/impl gotchas (summary)

- Set/def entities validate against meta defines ONLY → never hang non-meta predicate
  rels on a set entity; put the predicate on condition entities that point at the set.
- `Literal` isn't Hash/Eq → `ValueKey` wrapper using `f64::to_bits`.
- Per-type `cond-value-*` defs (literal type must match def target-type).
- Entity-target `cond-*`/`rel-predicate` defs: level `None` to avoid `LevelMismatch`
  when targeting def entities (defs are not sets).
- RedbStore shares `Indexes` → B's trait method is a one-line delegate on both backends.
- Intensional sets have no `meta.set` members → any code/test using `members_of_set`
  on a cat must switch to `resolve_members` / `Selection`.
- Keep deterministic output (sort) everywhere, matching `select.rs` convention.

## Suggested order

1. B (value index) — foundational, self-contained, both backends.
2. A (derive: intensional sets) — depends on B; wire select→resolve_members.
3. C (effective_members) — depends on resolve_members.
4. D (reify) — independent; can be done any time.
5. Consolidator (b) — depends on A (+B). Update the one test.
6. Design note (a).

Commit per feature. After each km-core feature: `cargo test` + `cargo clippy
--all-targets` in km-core. After consolidator: `cargo test` in the app crate (it
recompiles km-core via the path dep). Push both branches.
