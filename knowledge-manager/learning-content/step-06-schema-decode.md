# Step 6 — Decoding schema from raw entities

File: `src/schema.rs`. This is the read side of "schema is data": a `SetView`/`DefView`
is a projection of an ordinary entity's `meta.*` edges, decoded on demand.

## Understand

**A `DefView`/`SetView` is never stored — where does its data come from?**
- Decoded live from the entity's relationships. `def_view`/`set_view` read `rels_by_def`
  for each `DEF_META_*` and assemble a typed struct. Nothing typed sits on disk; only
  raw entities + `meta.*` edges do.

**Trace `def_view`: raw entity → typed view.**
1. `store.get(def_id)?` — fetch the entity.
2. `first_string(.., DEF_META_KIND)` must be `Some("def")`, else return `None`.
3. `first_entity(.., DEF_META_TARGET)` → optional allowed set.
4. `first_string(.., DEF_META_TARGET_TYPE)` (default `"string"`) → `parse_target_type`
   → `TargetType::Literal(..)` or `EntitySet(allowed)`.
5. `DEF_META_TARGET_LEVEL` → `Level::Instance/Schema` (or `None`).
6. `DEF_META_CARDINALITY` → `Card::Many` if `"many"` else `One`.
7. `DEF_META_REQUIRED` → bool (default false).
8. `DEF_META_CONSTRAINT` string → `parse_constraint` → `Range`/`Allowed`/`AllowedSet`.
9. `owning_set` via the *incoming* `meta.defines` edge.
- Result: a `DefView` with every field sourced from a `meta.*` edge.

**How does `resolve_def("Set.name")` turn a string into a `DefRef`?**
- `split_once('.')` → `(set_name, def_name)`.
- `by_name(set_name).first()` → the set's id.
- Walk that set's `meta.defines` targets; find the def entity whose `name == def_name`;
  return `DefRef::new(that_id)`.
- Purely structural — the qualified name is never stored, it's recomputed. Set scoping
  means composed sets never collide.

## Rust

- Enum variant shapes mixed in one enum: `Range { min, max }` (struct-like),
  `Allowed(Vec<Literal>)` (tuple), `AllowedSet(Id)`.
- `find_map(|r| ...)` — first element yielding `Some`, combining find + map in one pass.
- Function-as-value: `.map(str::to_string)` passes a function by name instead of a
  closure.
- `strip_prefix("range:")` → `Option<&str>` (prefix removed); `split_once("..")` →
  `Option<(&str,&str)>`; `.parse().ok()?` parse → `Option` → propagate `None` on
  failure.
- Turbofish `parse::<f64>()` names the target type when inference can't.
- `<S: Store + ?Sized>` so all these helpers accept `&dyn Store` too.

## Note

`is_set`/`is_def` are thin `meta.kind` checks reused throughout validation and views —
the kind discriminator is data, read the same way everywhere.
