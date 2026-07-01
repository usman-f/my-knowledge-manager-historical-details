# Implementation plan — relationship perspectives, properties, default value

Self-contained build plan. Rationale/derivation: `design/relationship-perspectives-and-edge-literals.md`
(read for the "why"; this file is the "how"). Priority: land all km-core changes, then fix the
dependent app `my-km-data-consolidator` (§Repo B).

## What km is (orientation)
- Uniform store: two building blocks — **entities** and **relationships** — where schema is itself
  data (`design/model.md`). `Entity { id, name, rels }`; `Relationship { rel_type: DefRef,
  target: Entity|Literal, qualifier: Option<Literal> }`, directed owner→target, stored on the owner
  (`crates/km-core/src/model.rs`).
- A **set** is an entity that `defines` relationship **defs**; an entity's assigned sets (`meta.set`)
  are its type. Reverse edges are a derived index (`store::incoming`). Bootstrap seeds the meta set +
  defs at ids 1..=10, `USER_ID_START = 1024` (`bootstrap.rs`, `id.rs`).
- Presentation (post-#47): a relationship shows the def's **bare name**; qualified `Set.name` is
  identity only (`view::def_label`).

## Target end-state
- **Property vs relationship** (vocabulary; storage stays uniform): entity→**literal** = *property*
  (uni-directional attribute); entity→**entity** = *relationship* (bi-directional).
- **Default value**: every entity may carry one canonical value literal `meta.value` (string by
  default); lookup `value_in(set, value)`. `name` may later derive from `{type} + value`.
- **Dual-named relationship**: a relationship def carries a forward name (its bare name) + an
  optional `meta.name-inverse`; the target side is presented under the inverse name.
- **Symmetric shape**: a relationship carries one **default property** (`qualifier`) + zero-or-more
  **named properties** — same shape as an entity's `meta.value` + properties.

## Ground rules
- Work on branch `claude/knowledge-manager-fundamentals-31iirk` (create from latest `main` if fresh).
- After each phase: `cd knowledge-manager && cargo test --all-features` must pass; commit per phase.
- Phases 1–3 are **additive** (no snapshot-shape change): reserved ids 11..1023 are free, so new meta
  defs do **not** move `USER_ID_START`; old user data is unaffected. Phase 4 changes the bincode
  `Relationship` shape → snapshots regenerate (km has **no committed `.km` fixtures**; the app
  regenerates its outputs, §Repo B).
- Keep existing `qualifier` and `value_entity` working throughout (no breaking removals).

---

## Repo A — my-knowledge-manager

### Phase 1 — default value (`meta.value`)  [G4]
Goal: a canonical value literal per entity + typed lookup within a type.
1. `crates/km-core/src/bootstrap.rs`
   - Add `pub const DEF_META_VALUE: Id = Id(11);` bump `MAX_BOOTSTRAP_ID` to `11`.
   - In `seed`: `store.put(meta_def(DEF_META_VALUE, "value", "string", "one", None));` and add
     `r(DEF_META_DEFINES, DEF_META_VALUE)` to the `meta` set's rels.
   - (Optional, only if typed values wanted now) support a target-type token `"literal"` = any
     literal: extend `schema::parse_target_type` + a `TargetType::LiteralAny` variant skipped in
     `validate::check_target`'s type check. Default plan keeps `value` string-typed.
2. `crates/km-core/src/schema.rs`
   - `pub fn entity_value<S>(store, id) -> Option<Literal>`: first `DEF_META_VALUE` literal, else
     `Some(Literal::String(entity.name))` (fallback).
   - `pub fn value_in<S>(store, set, value: &Literal) -> Option<Id>`:
     `store.owners_with_value(DEF_META_VALUE, value)` ∩ `members_of_set(set)`, first match. Keep the
     existing name-based `value_entity` untouched (both coexist).
3. `crates/km-edge-promote/src/lib.rs`
   - When interning a value-entity, also write `Relationship::new(DEF_META_VALUE, <value literal>)`
     on it (so the shared node carries its comparable literal, not just its `name`). Authorize
     `meta.value` on the target set's `defines` if not already (it is a universal meta def, so
     validation already allows it — confirm via `validate_all`).
4. Tests: `value_in(store,"Unit", &Literal::String("8".into()))` → the Unit-8 id; promoted entities
   carry `meta.value`; `validate_all` clean. Add to `km-core/tests` + a promote test.
5. Docs: `model.md` §Promoting a value / §Entity vs literal — record `meta.value` holds the
   comparable literal (fix the unimplemented "still holds its literal" claim).

### Phase 2 — dual-named relationship (`meta.name-inverse`)  [G1]
Goal: the target side of a relationship is presented under its own name.
1. `bootstrap.rs`: add `pub const DEF_META_NAME_INVERSE: Id = Id(12);` bump `MAX_BOOTSTRAP_ID` to
   `12`; seed `meta_def(DEF_META_NAME_INVERSE, "name-inverse", "string", "one", None)`; add
   `r(DEF_META_DEFINES, DEF_META_NAME_INVERSE)` to `meta`.
2. `schema.rs` `DefView`: add `pub name_inverse: Option<String>`; decode in `def_view` via
   `first_string(store, def_id, DEF_META_NAME_INVERSE)`.
3. `crates/km-core/src/view.rs`: add `fn def_label_inverse<S>(store, def) -> String` = the def's
   `name_inverse` else its bare name; use it wherever an **incoming**/reverse relationship is
   rendered (the forward/outgoing path keeps `def_label`). Mirror in
   `crates/km-edge-use-explore/src/lib.rs` for the `in_rels` rendering.
4. Scope: only entity-target (relationship) defs get an inverse; literal-target (property) defs
   ignore it. No validation change required (absent inverse ⇒ falls back to bare name).
5. Tests: give a def (e.g. `in-unit`) `name-inverse = "contains"`; assert the incoming render on the
   target entity shows `contains`; outgoing still shows `in-unit`.
6. Docs: `model.md` §Relationships (add inverse name); `decisions.md` (dual-named def entry).

### Phase 3 — property/relationship vocabulary + scoping  [G6]
Goal: name the entity/literal boundary; keep storage uniform.
1. `schema.rs`: `pub fn is_property_def(dv: &DefView) -> bool` = `matches!(dv.target_type,
   TargetType::Literal(_))`; `pub fn is_relationship_def(dv) -> bool` = the `EntitySet` case.
2. `km-edge-use-explore`: split an entity's rows into **Properties** (literal targets) vs
   **Relationships** (entity targets) using the predicates, so the explorer reflects the model.
   (Presentation only.)
3. No storage/validation change. Everything else is docs (§Phase 5).

### Phase 4 — native relationship properties (default + named)  [G2 / Feature D]  ⚠ heaviest
Goal: a relationship carries one default property (`qualifier`) + N named properties — the symmetric
shape. **Snapshot shape changes** (regenerates; no km fixtures). Fallback if deferring: use
`reify.rs` unchanged (no code) and stop after Phase 3.
1. `model.rs`: add `pub properties: Vec<(DefRef, Literal)>` to `Relationship` (keep `qualifier` as
   the default property). Constructors: keep `new`/`with_qualifier` (default `properties: vec![]`);
   add `with_properties(...)` / a builder `push_property(def, lit)`. Update the two struct-literal
   sites (`query.rs:86,122`) to set `properties: vec![]`.
2. Authorization model (symmetric with sets): a **relationship def** may `meta.defines` the property
   defs its instances can carry. Validation reads `defines_of(rel_def)` for allowed property defs.
3. `validate.rs`: for each `rel.properties` entry, check the property def is authorized on the
   rel's def, type/constraint (reuse `check_target`/`literal_satisfies`), and per-property-def
   cardinality (one/many).
4. `store/mod.rs`: extend the value index so an edge is findable by a property value — index each
   `(prop_def, value)` under the owner in `by_value` during `Indexes::index`/`unindex`; extend
   `Edge` to carry `properties` so `incoming` exposes them.
5. `view.rs` + `km-edge-use-explore`: render relationship properties as `name = value` on the rel
   row (default `qualifier` first, then named).
6. Persistence: bincode regenerates; run `km-cli load-example && validate`.
7. Tests: a relationship carrying `strength`+`order`+`weight` validates, renders, and is found by
   `owners_with_value(strength_def, …)`; one-property case uses only `qualifier` (no properties).
8. Decision checkpoint before starting: confirm the authorization model (step 2) or fall back to
   reify. Record the choice in `decisions.md` §Qualifier.

### Phase 5 — documentation sync (do in lockstep with each phase, not batched)
- `decisions.md` §Drop "properties": add note — storage fold stands (one `Target` enum); vocabulary
  un-folds to property (→literal) vs relationship (→entity).
- `decisions.md` §Qualifier: one **default** property + zero-or-more **named** properties (Phase 4);
  "one qualifier max" → "one default; more via named properties."
- `model.md` §Relationships / §Building blocks: property vs relationship split; symmetric shape
  `{ name(s), default value, properties[] }`; add `name-inverse` to the def triple.
- `model.md` §Namespacing/§Views: note #47 (bare-name presentation; qualified name = identity).
- `principles.md`: align any "no properties" wording with the vocabulary split.
- `README.md` (km-core bullet): mention property/relationship + `meta.value`.
- New `design/glossary.md`: entity≈node, relationship≈edge, property≈node/edge property,
  set≈label/class, qualifier≈default edge property, `meta.value`≈primary key/value.

### Acceptance (Repo A)
- `cargo test --all-features` green; `km-cli load-example && km-cli validate` clean.
- `value_in` resolves a member by value; incoming relationships render under the inverse name;
  (Phase 4) a multi-property relationship validates + is queryable.
- Design docs contain no statement contradicted by the code.

---

## Repo B — my-km-data-consolidator (follow-up)
Depends on km-core via **path** (`crates/data-consolidator/Cargo.toml`), so it compiles against the
updated km automatically — no version bump. Detail: `docs/impl-plan-km-fundamentals-followup.md`.
Summary of expected work:
- **Rebuild + test**: `cargo test` in the consolidator; fix any compile fallout. Constructors are
  preserved, so `equipment.rs` `with_qualifier(ie_has_record, …, db)` keeps working; the `db`
  qualifier is now the edge's **default property** (semantics unchanged).
- **Adopt `meta.value`**: promoted value-entities (`Unit`, `ProcessVariable`, roles) now carry
  `meta.value`; switch any name-based value lookups to `value_in` where a typed/canonical lookup is
  wanted (optional; `value_entity` still works).
- **Optional**: migrate `ie-has-record`'s per-member `db` from the single qualifier to a named
  relationship property if a second edge fact is needed (Phase 4); not required otherwise.
- **Regenerate outputs**: re-run the pipeline to refresh `out/*.km` and any snapshots; run the
  review/export tests.
- **Docs**: no domain-doc change unless a doc asserts "everything is a relationship"; then align to
  the property/relationship split.

## Suggested execution order
Phase 1 → 2 → 3 (each: implement, test, doc-sync, commit) → Phase 4 (checkpoint, then implement) →
Repo B (rebuild, adopt, regenerate, test). Stop-after-Phase-3 is a valid milestone (reify covers
multi-property in the interim).
