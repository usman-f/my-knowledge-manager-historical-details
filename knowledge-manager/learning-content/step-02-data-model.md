# Step 2 — The data model (entities & relationships)

File: `src/model.rs`.

## Understand

**Why is `Target = Entity(Id) | Literal(...)` "the central design boundary"?**
- It *is* the entity-vs-literal distinction, made one exhaustive enum:
  - `Entity` targets are shared, referenceable, knowledge-bearing concepts — free
    dedup, reverse queries, can attach their own facts (units, species, statuses).
  - `Literal` targets are inert, instance-specific scalars (measurements, dates, text).
- The whole "drop properties" decision rests here: a former property is just a
  relationship whose target is one or the other. Validation, views, and embedding
  rendering all branch on this single enum (RDF object-property vs datatype-property in
  one type).

**Why can `Relationship::new` accept both an `Id` and a `DefRef` for `rel_type`?**
- The parameter is `rel_type: impl Into<DefRef>`. `From<Id> for DefRef` exists (line
  68), and `DefRef` trivially converts into itself.
- So callers pass a bare `Id` (the common case) or a `DefRef`; `.into()` normalizes.

**What does `rels_of` return, and why does nothing happen until you iterate?**
- Returns `impl Iterator<Item = &Relationship>` — a lazy `filter` adapter, not a
  collection.
- Iterator adapters do no work until *consumed* (a `for` loop, `.collect()`,
  `.count()`, `.next()`). Laziness lets the caller decide: count them, take the first,
  collect into a Vec — without `rels_of` allocating anything.
- `move` makes the closure own the captured `def` (an `Id`, `Copy`), so the returned
  iterator doesn't borrow a local that goes out of scope.

## Rust

- Generic impl `impl<L: Into<Literal>> From<L> for Target`: one impl covers every type
  convertible to a `Literal` (`&str`, `i64`, `bool`, …).
- `impl Trait` in argument position (`impl Into<DefRef>`) = shorthand for a generic
  param with that bound; ergonomic constructors lean on it.
- Ownership: enum variants own their data (`Literal::String` owns a `String`), so
  `&str` must be `.to_string()`-ed before storage.
- `&mut self` + returning `&mut Self` enables method chaining (`e.push(a).push(b)`).
- "field init shorthand": `Self { id, name, rels }` when locals match field names.

## Try it — count relationships filling a def

```rust
fn count_def(e: &Entity, def: Id) -> usize {
    e.rels_of(def).count()
}
```

`rels_of` yields the lazy filtered iterator; `.count()` consumes it, allocating
nothing.
