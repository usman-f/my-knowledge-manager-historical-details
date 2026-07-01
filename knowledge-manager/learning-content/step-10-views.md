# Step 10 — Derived views (rendering)

File: `src/view.rs`. Nothing here is stored — every view is recomputed from edges.

## Understand

**Confirm nothing is stored.**
- Each function takes `&S: Store` and returns a `String` built on the fly:
  - `by_type` walks `store.ids()`, filters out set/def entities, keeps those whose
    `effective_sets` contain the target set.
  - `schema_view` iterates set entities → `set_view` → `def_view` per def.
  - `data_view` resolves `effective_sets(id)` + labels each relationship.
  - `neighborhood` reads outgoing `e.rels` + incoming `store.incoming(id)`.
- No caching, no stored projection — consistent with "views are derived" (cache hot ones
  *above* this layer if needed).

**Difference between the four views.**
- `schema_view` — the whole schema: every set, its build-on parents, its qualified defs
  with target-type/cardinality/required/constraint.
- `data_view` — one entity: its effective sets + every relationship as
  `def [qualifier] -> target-label`.
- `neighborhood` — one entity: outgoing *and* incoming edges (incoming via the reverse
  index, carrying the qualifier).
- `by_type` — all *data* entities of a set (following build-on), excluding schema nodes.

## Rust

- `if` is an *expression* (no ternary): `if e.name.is_empty() { id.to_string() } else
  { e.name }` is the body of `.map(..)`; both branches yield `String`.
- Build owned text with `String::from`, `push_str(&str)`, `format!` (returns a String).
  `&format!(..)` borrows the temporary as the `&str` `push_str` wants.
- `match` as an expression to unwrap-or-skip: `let sv = match set_view(..) { Some(v) =>
  v, None => continue };` (the newer shorthand is `let ... else`).
- `Option::as_ref().map(..).unwrap_or_default()` to render an optional qualifier as
  `" [PV]"` or `""`.

## Note

`label`/`def_label`/`target_label` are the shared formatting helpers: an entity shows
its name (or id if unnamed), a def shows its qualified `Set.name`, a target shows the
entity label or the literal's `Display`.
