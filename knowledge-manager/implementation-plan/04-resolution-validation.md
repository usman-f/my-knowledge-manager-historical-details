# Resolution & Validation

## Schema decode (`schema.rs`)
Typed views over raw entities; never stored, recomputed/cached.
```rust
pub struct SetView   { pub id: Id, pub name: String,
                       pub builds_on: Vec<Id>,   // meta.set targets (schema-level)
                       pub defines: Vec<Id> }     // meta.defines targets
pub struct DefView   { pub id: Id, pub name: String, pub owning_set: Id,
                       pub target_type: TargetType,   // Literal(LiteralType) | EntitySet(Option<Id>)
                       pub target_level: Level,       // Instance | Schema
                       pub cardinality: Card,         // One | Many
                       pub required: bool,
                       pub constraint: Option<Constraint>,
                       pub qualifier: Option<(LiteralType, Option<Constraint>)> }
```
- `SetView::from(entity)`: read `meta.set` (build-on) and `meta.defines` rels.
- `DefView::from(entity, owning_set)`: read `meta.*` rels; `owning_set` found via reverse index on `meta.defines`.
- `qualified_name(def) = set.name + "." + def.name`.

## Effective sets — transitive closure (`resolve.rs`)
```rust
pub fn effective_sets(store, e: Id) -> BTreeSet<Id>
```
Algorithm:
1. seed = direct `meta.set` targets of `e` (+ universal set if any).
2. BFS/DFS: for each set, add its `meta.set` (build-on) targets.
3. cycle-guard with visited set (build-on may be mis-authored cyclic — terminate, don't loop).
Result is the union per design ("multiple sets compose as a union"). Cache by entity id, invalidate on `put`.

Worked check (from `example.md`): `02FIT100` direct {Transmitter, ProcessSignals} ⇒ +Instrument ⇒ {Transmitter, Instrument, ProcessSignals}. Unit test asserts exactly this.

## Definition lookup
```rust
pub fn defs_for(store, e: Id) -> Vec<DefView>   // all defs from effective sets
pub fn resolve_def(store, "Set.name") -> Option<DefRef>
```
- `defs_for` = flat-map `meta.defines` over effective sets, deduped by qualified name.

## Validation (`validate.rs`)
```rust
pub fn validate_entity(store, e: Id) -> Vec<Violation>
pub fn validate_all(store) -> Vec<Violation>
```
Per entity, for each relationship:
1. **closed-def**: `rel.rel_type.def` must resolve to a def in `defs_for(e)`. Else `UndefinedRelationship`. (Closed on definitions.)
2. **target kind**: `Target::Entity` requires `target_type = EntitySet`; `Target::Literal` requires `target_type = Literal(t)` with matching `lit_type`. Else `TargetTypeMismatch`.
3. **target level**: if `EntitySet`, fetch target entity; `Instance` ⇒ target must NOT be a set (`meta.kind != "set"`); `Schema` ⇒ target MUST be a set. Else `LevelMismatch`.
4. **allowed set**: if def names an allowed set, target's effective sets must include it. Else `OutOfSet`.
5. **constraint**: range/allowed-values checked against literal (target or qualifier). Else `ConstraintViolation`.
6. **qualifier**: present only if def allows; type + constraint match. Else `QualifierError`.
Per definition, across the entity's rels of that def:
7. **cardinality**: `One` ⇒ ≤1 instance; **required** ⇒ ≥1. Else `CardinalityError` / `MissingRequired`.

Open-at-value rule: absence of a non-required relationship is **never** an error (absence ≠ false). Only `required` defs trigger `MissingRequired`.

```rust
pub enum Violation {
  UndefinedRelationship{owner:Id, def:Id},
  TargetTypeMismatch{..}, LevelMismatch{..}, OutOfSet{..},
  ConstraintViolation{..}, QualifierError{..},
  CardinalityError{..}, MissingRequired{..},
}
```

## Validation mode
- `put` is permissive by default (store can hold work-in-progress); `validate_*` is an explicit pass (lint/commit-gate). Optional strict mode flag rejects on `put`.
- `set:meta` and `def:meta/*` are skipped (pre-validated fixed point) to avoid bootstrap regress.
