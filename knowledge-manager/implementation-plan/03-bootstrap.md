# Bootstrap — Meta Set & Primitives

Seed written once into an empty store (`bootstrap::seed(&mut store)`). All bootstrap entities are ordinary `Entity` rows; the meta set is defined in terms of itself (fixed point).

## Reserved ids
Low, fixed `Id`s so seed + tests are deterministic:
```
1  set:meta
2  def:meta/kind
3  def:meta/target-type
4  def:meta/target-level
5  def:meta/cardinality
6  def:meta/required
7  def:meta/constraint
8  def:meta/set
9  def:meta/defines
10 def:meta/target
```
`IdGen` starts user ids at e.g. 1024 (room for future bootstrap entities).

## Primitive literal types
Not entities — they are the `LiteralType` enum (`number/string/bool/date`) baked into code (design: "literal types will likely just be the host language's primitives"). Definitions reference them by enum tag in `meta.target-type`, not by id.

## Meta set, as stored
```
set:meta  name="meta"
  meta.kind -> "set"
  meta.defines -> def:meta/kind
  meta.defines -> def:meta/target-type
  meta.defines -> def:meta/target-level
  meta.defines -> def:meta/cardinality
  meta.defines -> def:meta/required
  meta.defines -> def:meta/constraint
  meta.defines -> def:meta/set
  meta.defines -> def:meta/defines
  meta.defines -> def:meta/target
```
Each `def:meta/*` entity:
```
def:meta/kind        name="kind"        meta.kind->"def"  meta.target-type->String
def:meta/target-type name="target-type" meta.kind->"def"  meta.target-type->String (enum: literal|entity-set)
def:meta/target-level name="target-level" meta.kind->"def" meta.target-type->String (enum: instance|schema)
def:meta/cardinality name="cardinality" meta.kind->"def"  meta.target-type->String (enum: one|many)
def:meta/required    name="required"    meta.kind->"def"  meta.target-type->Bool
def:meta/constraint  name="constraint"  meta.kind->"def"  meta.target-type->String (encoded; see below)
def:meta/set         name="set"         meta.kind->"def"  meta.target-type->entity-set, level=schema
def:meta/defines     name="defines"     meta.kind->"def"  meta.target-type->entity-set, level=schema
def:meta/target      name="target"      meta.kind->"def"  meta.target-type->entity-set, level=schema
```

## Schema-as-data encoding choices
The meta vocabulary is itself relationships on def entities. To avoid an infinite regress of "a def to describe meta.kind", bootstrap def attributes (`meta.kind`, `meta.target-type`, etc.) are stored as relationships whose `DefRef` points at the corresponding `def:meta/*` — i.e. the meta set's own definitions describe the meta set. The loop closes because `seed()` writes these rows directly, and `validate()` treats `set:meta` as pre-validated (trust the fixed point, then validate everything built on it).

## Constraint encoding
`meta.constraint` target is a `String` literal holding a small encoded form (parsed by `schema.rs`):
- `range:<min>..<max>` (numbers/dates)
- `allowed:<lit>,<lit>,...` (enumerated literals)
- `set:<setName>` (allowed entity set for entity targets)
Phase 2 option: promote constraints to structured entities; encoding chosen now to ship Level 0 without a second schema language.

## Universal set hook
A set assigned to all entities (design §"universal set") is recorded as a well-known set id; `resolve.rs` injects it into every entity's effective sets. Not seeded until a concrete universal set (beyond meta) exists.

## Seed test
`seed()` then `validate_all()` must pass with zero errors; round-trip `seed → dump → reload → dump` is byte-identical.
