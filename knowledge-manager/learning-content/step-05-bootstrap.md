# Step 5 — Bootstrap: the self-defining meta schema

File: `src/bootstrap.rs`.

## Understand

**How is the `meta` set "defined in terms of itself"? Which ids are reserved?**
- `meta` is the entity at `SET_META = Id(1)`. It has `meta.kind = "set"` and nine
  `meta.defines` edges, one pointing at each of the nine `DEF_META_*` defs
  (`Id(2)..=Id(10)`) — and crucially includes `meta.defines -> DEF_META_DEFINES` and
  `-> DEF_META_KIND`.
- So the relationships that *describe* defs (`kind`, `defines`, …) are themselves
  declared by defs that the meta set holds: a fixed point. Each meta def is built with
  `meta_def`, which uses `meta.kind`/`meta.target-type`/`meta.cardinality` — the very
  defs being defined.
- Reserved ids: `1..=10` (`MAX_BOOTSTRAP_ID`). User data starts at `USER_ID_START =
  1024`, leaving room.

**Why does each `DEF_META_*` constant correspond to a field of a definition?**
- Schema is data. A `DefView` (Step 6) is decoded from a def entity's `meta.*` edges:
  `kind`, `target-type`, `target-level`, `cardinality`, `required`, `constraint`,
  `target`. Plus structural `set` (assignment/build-on) and `defines` (containment).
  Each field ↔ one meta def.

**Why must bootstrap ids be skipped by the validator (`is_bootstrap`)?**
- The meta core is circular/self-defining: it provides the very defs used to validate.
  Running validation over it would either regress infinitely or flag the fixed point
  against rules it bootstraps. It's treated as a pre-validated axiom.

## Rust

- `pub const SET_META: Id = Id(1);` — compile-time constants, SCREAMING_SNAKE_CASE,
  explicit type.
- Private free fn `r(...)` = module-local helper (short name keeps dense seed data
  readable); not visible outside the module.
- `let mut rels = vec![...]` then conditional `.push()` — `mut` opt-in; `vec!` macro
  builds a `Vec`.
- `if let Some(lvl) = level { ... }` — run a block only when the `Option` is `Some`.
- `seed<S: Store>(store: &mut S)` — generic over any backend, mutable borrow (no
  ownership taken).

## Map of the nine meta defs

| Id | const | name | encodes |
|----|-------|------|---------|
| 2 | `DEF_META_KIND` | kind | set / def / (data: none) |
| 3 | `DEF_META_TARGET_TYPE` | target-type | literal type or `entity-set` |
| 4 | `DEF_META_TARGET_LEVEL` | target-level | instance / schema |
| 5 | `DEF_META_CARDINALITY` | cardinality | one / many |
| 6 | `DEF_META_REQUIRED` | required | bool |
| 7 | `DEF_META_CONSTRAINT` | constraint | range / allowed / set |
| 8 | `DEF_META_SET` | set | assign a set; set→set build-on |
| 9 | `DEF_META_DEFINES` | defines | set → its defs |
| 10 | `DEF_META_TARGET` | target | def → allowed entity set |
