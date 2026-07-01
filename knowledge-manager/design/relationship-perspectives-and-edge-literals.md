# Relationship perspectives + edge literals — alignment & gaps

Planning note (no code change). Reviews seven fundamentals against km-core and maps gaps.
Refs: `model.rs`, `store/mod.rs`, `schema.rs`, `view.rs`, `reify.rs`, `derive.rs`, `select.rs`;
`decisions.md` §Qualifier/§Drop properties, `selection-and-derived-sets.md` §D,
`model.md` §Relationships. Grounding example: a refinery process **Unit 8** and its equipment.
Rebased onto `main` incl. #47 (relationships presented by **bare def name**, not qualified
`Set.name`; qualified name is identity only — `view::def_label`). This *reinforces* G1/G3 below.

## Fundamentals under review
1. **Bi-directional link.** A relationship to another entity has two endpoint perspectives:
   what the owner calls the target, and what the target calls the owner. Both sides must know
   what the other is *from its own perspective* (owner-side name + target-side name).
2. **Global (third) perspective.** How an outside/global observer refers to and accesses an
   entity — independent of either endpoint. May coincide with an endpoint perspective or not.
3. **Named link literals.** A link may carry one or more *named* literals describing the link
   itself (importance, order, weight; e.g. `Friend.strength = 0.8`).
4. **Fast access.** O(1)-ish paths to (a) all entities of a type (global perspective), and
   (b) a specific entity (then its related entities).
5. **Default value.** Every entity has one canonical value literal (string preferred) — its
   handle for lookup within a type: "in set `Unit`, find the member with value `8`." This is the
   entity's own global-perspective identity (fundamental 2b), made a first-class field.
6. **Property vs relationship.** The entity/literal target boundary *is* the property/relationship
   boundary. **Relationship** = entity→entity (bi-directional, dual-named, F1). **Property** =
   entity→literal (uni-directional attribute; a literal has no perspective, no reverse name). F1
   and F3 apply only to relationships.
7. **Symmetric shape.** A relationship carries its own facts like an entity does: **one default
   property** (the canonical qualifier literal) + **zero-or-more named properties**. So entity and
   relationship share the shape `{ dual name(s), default value, properties[] }`.

## Refinery worked example
Sets (types) and members:
- `Unit` — value-entities `Unit 8`, `Unit 9`, … (unit numbers as shared nodes).
- `IdentifiedEquipment` (IE) — `Pump P-101`, `Controller 08FIC100`, … the tag-lookup hubs.
- `EquipmentRole`, `ProcessVariable` — shared value-entity sets.

Stored links today (owner → target), all **equipment-side**:
```
Pump P-101        in-unit            -> Unit 8          (IE points at the unit)
Pump P-101        has-role           -> Pump            (EquipmentRole)
08FIC100          measures-variable  -> Flow            (ProcessVariable)
08FIC100          ie-has-record ["SAP"] -> <source rec> (many, qualified by source DB)
```
Question the fundamentals ask of this data:
- Get **all units** → global perspective on the `Unit` set.
- Get **Unit 8**, then **its equipment** → specific entity + its relations.
- Unit 8's own name for that equipment (its side of `in-unit`).
- Attach *several* facts to one `in-unit` link (role-in-unit, install date, criticality).

## Current model (facts)
- `Entity { id, name, rels }`; `Relationship { rel_type: DefRef, target: Entity|Literal,
  qualifier: Option<Literal> }`. Directed, stored **only on the owner** (`model.rs`).
- Reverse is **derived**, not stored: `Indexes.reverse` → `store.incoming(target)` yields
  `Edge { owner, def, qualifier }` (`store/mod.rs`). Both endpoints are therefore *reachable*
  from one stored edge; `dangling_targets` relies on this ("both endpoints know the link").
- Global access: `members_of_set(set)` (O(1), direct `meta.set` members), build-on-aware
  `effective_members` (`select.rs`), intensional sets (`derive.rs`); `by_name` (O(1)),
  `value_entity(set,value)` to disambiguate same-named members (`schema.rs`).
- One qualifier per edge, its type declared on the def (`decisions.md` §Qualifier).
  `reify.rs` is the escape hatch: mint a `ReifiedEdge` entity carrying arbitrary named
  property rels, findable from both endpoints and by value (Feature B index).

## Alignment matrix
| Fundamental | Mechanism today | Status |
|---|---|---|
| 1 owner-side name | `rel_type` def (qualified `Set.name`) | ✅ |
| 1 target-side name | none — reverse index re-uses the **same** def name | ❌ G1 |
| 1 both-sides-know | link is *reachable* from both ends (reverse index) | ⚠️ reachable, not *named* both ways |
| 2 all-of-type | `members_of_set` / `effective_members` / intensional | ✅ |
| 2 specific entity | `by_name` + `value_entity(set,value)` | ✅ |
| 2 global link handle | #47: relationships presented by def's **bare name**; qualified `Set.name` = identity only | ⚠️ G3 (one presented name, no inverse) |
| 3 one named literal | `qualifier` (name lives on the def, not the instance) | ⚠️ single, def-named |
| 3 many named literals | `reify.rs` (heavier) or deferred native multi-qualifier | ❌ G2 |
| 4 fast type/entity access | index-backed (above) | ✅ |
| 5 default value (string) | `name` field is de-facto the value; `value_entity(set,value)` matches on it | ✅ works, but `name` is overloaded (display + lookup key) |
| 5 default value (typed) | no held literal — promote interns **by name** only; sort/range on value is lexical | ❌ G4 |

## Gaps

### G1 — no target-side (inverse) name; no inverse pairing
Stored: `Pump P-101 --in-unit--> Unit 8`. `incoming(Unit 8)` returns the same edge labeled
`in-unit`; Unit 8 has **no** vocabulary of its own for that member (e.g. `contains` /
`has-equipment`). Two current work-arounds, both lossy:
- rely on the derived reverse index → single name, no target-side term; or
- store a second independent edge `Unit 8 --contains--> P-101` → the core does **not** know
  the two defs are inverses, co-maintains nothing, validates nothing, can drift.
Requirement (fundamental 1) is *both sides named*: today only the owner side is named.

### G2 — only one named literal per link (native)
`qualifier: Option<Literal>` → at most one. Multiple facts on one link
(role-in-unit + install-date + criticality, or `strength` + `order` + `weight`) have nowhere
native to go. `reify.rs` handles it but costs a stand-in entity + an extra hop per edge, and is
an application construct, not a link feature. Native multi-qualifier = Feature D
(`selection-and-derived-sets.md`), deferred: breaks the bincode snapshot shape, touches
`model.rs` + `validate` + persistence + every consumer.

### G3 — qualifier is not instance-named; single presented link name
The single qualifier's role/name is declared on the **def**, not per instance, so "which fact
is this literal" is implicit (G2/F7 resolve this via named relationship properties). Separately,
after #47 a relationship is presented by the def's **bare name** — a set-neutral presentation, and
the qualified `Set.name` is a clean *identity* handle — but it is still **one** name (the forward
side). The inverse-side presentation is exactly what G1 adds. Minor once G1 lands.

### G4 — no dedicated default value; `name` is overloaded, and it is string-only
The entity `name` already serves as the canonical value (Unit 8's `name` is `"8"`;
`value_entity("Unit","8")` finds it O(1) via the name index + set filter). Two problems:
- **Overloaded field.** `name` is simultaneously the display label *and* the lookup key. Fine
  when they coincide (`"8"`), wrong when they must differ (display `"Crude Feed Pump"` vs value
  `"P-101"`). No slot separates them.
- **String-only, no typed value.** Promotion interns value-entities **by name** (a string); there
  is no held typed literal. So "units 5..10" or numeric/date ordering within a type is **lexical
  on the name**, not numeric — despite `model.md`/`decisions.md` promising a promoted value-entity
  "still holds its literal so it stays comparable." That literal is not actually stored today.

## Decisions
- **G1 → dual-named definition** (chosen). The relationship **definition** carries *both*
  perspective names; a single stored directed edge self-describes both sides through its def.
- **G2 → relationship properties, one default + many named** (chosen, revised). Target model:
  a relationship carries one **default property** (the current single `qualifier`) + zero-or-more
  **named properties** — symmetric with an entity's `meta.value` + properties (F7). Interim
  implementation: `reify.rs` for the multi-property case (no core change); native multi-property
  (Feature D) is the target once a consumer needs it. Supersedes the earlier "reify only" lean —
  reify stays the *interim*, native is the *goal*.
- **G6 → property/relationship split, uniform storage (chosen).** Treat entity→literal as a
  **property** and entity→entity as a **relationship**; F1/F3/G1 apply to relationships only.
  Keep the uniform `Target::Entity|Literal` storage — the split is conceptual + def-level
  (`TargetType` already discriminates) + which-fundamentals-apply, **not** a `model.rs` rewrite.
- **G4 → optional `meta.value` def** (chosen, C2). Canonical value literal (default string, typed
  allowed); falls back to `name` when absent; value-indexed for typed `value_in(set,value)`.
  **Trajectory:** `value` becomes the canonical identity and `name` a *derived/display* field —
  candidate rule `name := {global type/set name} + " " + value` (e.g. `Unit` + `8` → `"Unit 8"`).
  Aligns with fundamental 2 (the global perspective is exactly type + value). Not built now; the
  optional `meta.value` def is the enabling step, and `name` stays authored until the derive rule
  lands so nothing breaks.

## Options considered

### G1 — bidirectional / inverse names
- **A1 Dual-named def (CHOSEN).** One def entity holds both directional names — a **side-A name**
  (owner→target, e.g. `in-unit`) and a **side-B name** (target→owner, e.g. `contains`) via two
  new def fields (`meta.name-forward` / `meta.name-inverse`; the def's bare `name` — already the
  forward presentation after #47 — stays the side-A/global handle, so `name-forward` may just be
  that bare name and only `name-inverse` is genuinely new). The edge stays a single directed
  `Relationship` referencing that def, so
  both endpoints and both names come from **one** stored fact — "the edge contains all the
  information: link + name for side A and side B." Reads: `incoming(Unit 8)` renders under the
  side-B name (`contains`); no second edge, nothing to drift. Cost: 2 bootstrap def-fields +
  inverse-name-aware read in `view`/`neighbors`; **no `model.rs` or persistence change** (names
  live on the def entity, not the edge instance).
  - vs. two defs paired by `meta.inverse`: rejected — splits one concept across two def entities.
- **A2 Auto-materialize the inverse edge.** On `relate`, also write the paired reverse edge. Two
  stored truths → drift risk (the `both-independent` trap in `selection-and-derived-sets.md`).
  Rejected.
- **A3 Name the reverse index only.** Per-def display label on `incoming`, one direction. Rejected
  — one vocabulary, does not name both sides.

### G2 — multiple named literals on a link
- **B1 Reify when >1 fact (CHOSEN).** Use `reify.rs`; keep single `qualifier` for the 1-fact
  common case. Zero core change. Extra hop per multi-fact edge.
- **B2 Native multi-qualifier (Feature D).** `qualifier: Option<Literal>` →
  `qualifiers: Vec<(DefRef, Literal)>` (named per instance, validated by def). Cleanest for the
  `Friend.strength`/`order`/`weight` case; largest blast radius (snapshot shape + every
  consumer). Deferred until a second consumer needs it.

### G4 — default value
- **C1 Formalize `name` as the value (lightest).** Document that `name` *is* the default value;
  `value_entity(set,value)` is the lookup. Zero code. Keeps the overload (display == key) and
  stays string-only — no typed sort/range. Fits the stated `Unit 8` example exactly.
- **C2 Optional `meta.value` literal def (recommended).** A bootstrap def `value` (literal, `one`,
  default type string; typed allowed) holding the canonical value; when absent, fall back to
  `name`. Reuse the existing value index (Feature B, `owners_with_value`) so
  `value_in(set, value) = owners_with_value(value_def, lit) ∩ members_of_set(set)` — typed and
  O(result). Decouples display `name` from the lookup key and finally stores the comparable
  literal `model.md` promised. Promotion writes `value` (not just name) when interning. Cost: one
  bootstrap def + a `value_in` helper + a promote tweak; no `model.rs`/snapshot change.
- **C3 Typed-mandatory value.** Force a typed literal per entity. Rejected — over-constrains the
  string-preferred common case and every plain entity would need a value.

### G3
- Subsumed by G1: the dual-named def already gives two names; the def's bare `name` is the
  neutral/global handle. Add a distinct global link label only if a concrete need appears.

## F6 — property vs relationship (the entity/literal boundary)
The new fundamentals presuppose two entities, so they only apply to entity-target facts:
- **F1 bi-directional / dual-named** is vacuous for a literal. `Pump P-101 --design-pressure->
  150`: what does `150` call P-101? Nothing — a literal has no identity, no reverse edge, no
  second name. G1's `name-inverse` is meaningless for a literal-target def (it can only ever have
  the forward name) — G1 is already "entity-targets only."
- **F3 link literals** annotate an edge between two things; a property *is* the literal.
- **F5 default value** (`meta.value`) *is itself* an entity→literal fact — the prototypical
  property. And because promotion turns any share-worthy value into an entity that carries its own
  value, the residual literals are exactly the inert instance scalars = properties.

**Test that sorts any fact:** does the target have a perspective and a reverse name? No → property.
Yes → relationship.

### P&ID drawing example (`PID-2100-08`)
| Fact | Target | Kind | Reverse perspective? |
|---|---|---|---|
| drawing-number `"PID-2100-08"` | literal | **property** (its `meta.value`) | no |
| revision `"C"`, sheet `3`, issue-date `2024-11-02` | literal | **property** | no |
| depicts ↔ shown-on `Pump P-101` | entity | **relationship** | yes — "appears on drawings" |
| covers-unit ↔ depicted-on `Unit 8` | entity | **relationship** | yes |
| supersedes ↔ superseded-by `PID-2100-07` | entity | **relationship** | yes |

Promotion still governs the gray zone: if revisions need comparison/approval/query, `"C"` becomes
a `Revision` entity (`meta.value = "C"`) and `has-revision` a real bi-directional relationship —
F5 is what lets the promoted entity keep `"C"` as its value.

`Unit 8` sorts identically: `unit-number "8"`, `commissioned 2019-06-01`, `design-capacity 50000`
are properties; `contains`/`feeds`/`operated-by`/`depicted-on` are relationships. A **quantity**
(`design-capacity 50000 bbl/day`) is one fact with both halves: literal magnitude (property) +
unit-of-measure entity (relationship) — the split coexists within one fact.

### Consistency with prior decisions
Partial reversal of `decisions.md` §Drop properties — but only of the *vocabulary*, not the
storage. The original simplicity win (one `Target` enum, one def kind) is retained: `TargetType`
already discriminates literal vs entity-set, so "property" and "relationship" are a **typed lens**
over the uniform store (same pattern as sets/defs/views). §Keep literals is untouched — literals
remain the ground floor; we only rename the entity→literal link a "property" and note it is
uni-directional.

## F7 — symmetric shape (entity ≙ relationship)
Both carry a default value + a bag of named properties:

| Aspect | Entity | Relationship |
|---|---|---|
| identity | `id` | (`subject`, `def`, `object`) — or a reified id |
| name(s) | `name` (derivable, F5 trajectory) | def's dual names (side-A / side-B, G1) |
| default value | `meta.value` (one literal) | `qualifier` (one literal) = the **default property** |
| named properties | entity properties (→literal, many) | relationship properties (→literal, many, G2) |
| relationships | entity→entity edges | (a reified relationship can itself relate to entities) |

Consequence: the "relationship properties" of G2 are the exact analogue of entity properties, and
`reify.rs` already realizes this by modeling an edge **as an entity** (so it inherits the entity
shape wholesale). Native multi-property on `Relationship` (Feature D) is the same shape without the
stand-in entity; reify is the interim, native the goal. A one-property relationship uses only the
default `qualifier` — no reification, no cost.

## Documentation to update (when these land)
Keep the design corpus from contradicting the new fundamentals. Per-file edits:
- **`decisions.md` §Drop "properties"** — add a follow-up note: the *storage* fold stands
  (one `Target` enum), but the **vocabulary** is un-folded — entity→literal = **property**
  (uni-directional), entity→entity = **relationship** (bi-directional). Point to F6 here.
- **`decisions.md` §Qualifier** — revise: a relationship carries one **default property** (the
  qualifier) **+ zero-or-more named properties** (F7/G2); "one qualifier max" becomes "one
  *default*; more via named properties (reify interim, native goal)."
- **`model.md` §Relationships** — split the prose into **relationship** (→entity, dual-named,
  properties) vs **property** (→literal, one name, uni-directional); state the symmetric shape
  `{ name(s), default value, properties[] }` (F7). Add `name-inverse` alongside the directed
  `(type, target, qualifier?)` triple.
- **`model.md` §Building blocks** — "two building blocks" now read as entity + relationship where
  a *property* is the entity→literal case of a relationship (name the property/relationship lens).
- **`model.md` §Promoting a value / §Entity vs literal** — record that the comparable literal is
  stored as the promoted entity's **`meta.value`** (G4), not merely its `name`; fix the current
  "still holds its literal" claim which no code implements.
- **`model.md` §Namespacing / §Views** — note #47: relationships are *presented* by bare def name;
  qualified `Set.name` remains identity.
- **`principles.md`** — if it restates "no properties," align it with the F6 vocabulary split.
- **`README.md` (km-core bullet)** — mention property vs relationship + `meta.value` when built.
- **New `design/glossary.md`** (from the prior turn) — entity≈node, relationship≈edge,
  property≈node/edge property, set≈label/class; single home for the vocabulary mapping.
- **`my-km-data-consolidator`** — no doc change required (domain examples already fit); revisit
  only if IE/record docs assert "everything is a relationship."

## Implementation ordering (follow-up plan)
1. **G1 via A1 (dual-named def).** Add `meta.name-forward` / `meta.name-inverse` def-fields
   (bootstrap), decode into `DefView`, render the inverse name in `view`/`neighbors`/`incoming`.
   No `model.rs` or snapshot change. Realizes fundamental 1; Unit 8's `contains` view becomes
   first-class off the single stored `in-unit` edge.
2. **G2 — relationship properties (reify interim → native goal).** One default property
   (`qualifier`) needs nothing new. Multi-property: `reify.rs` now; native multi-property
   (Feature D) when a consumer needs it — the F7-symmetric target.
6. **G6 — property/relationship vocabulary split.** Doc + validation scoping only (F1/F3/G1 gated
   to entity-target defs); uniform storage unchanged. No `model.rs`/snapshot change.
7. **Docs sync** (the list above) — do this in lockstep with each feature so `decisions.md` /
   `model.md` never contradict the code. Cheapest done per-feature, not batched at the end.
3. **G4 via C2 (optional `meta.value` def).** Bootstrap `value` def; `value_in(set,value)` helper
   over the existing value index; promote writes `value`; fall back to `name` when absent. No
   `model.rs`/snapshot change. Later (separate step): make `name` a derived field
   `{type name} + value`, at which point `value` is the sole authored identity.
4. **G3** deferred.
5. Fundamentals 2 and 4 already met — no work; fundamental 5's string case already works via
   `name`/`value_entity` (G4 adds the typed value + display/lookup split).
