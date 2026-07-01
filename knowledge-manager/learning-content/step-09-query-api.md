# Step 9 — Query/Graph API & change notifications

File: `src/query.rs`. The read/write/traversal surface that processing layers build on
without touching the symbolic core.

## Understand

**`ChangeSink` uses `dyn`; `EmbeddingIndex<E>` uses generics — why each?**
- Sinks are a heterogeneous, runtime-built list: `sinks: Vec<Box<dyn ChangeSink>>`. The
  `Graph` can't know each sink's concrete type, so it uses dynamic dispatch (trait
  object). `Box` is required because `dyn ChangeSink` is unsized; the heap pointer is the
  uniformly-sized Vec element.
- `on_put(&mut self, store: &dyn Store, id)` takes the store as `dyn` too, so a sink
  isn't parameterized by the store's concrete type.
- `EmbeddingIndex<E: Embedder>` (Step 15) instead monomorphizes per embedder — zero-cost
  static dispatch, because the concrete embedder is known where it's constructed.

**In `notify_put`, why destructure `self` into `{ store, sinks }` first?**
- To borrow two fields disjointly: iterate `sinks` mutably (`iter_mut`) while passing
  `&store` to each sink. You can't take two borrows of the *whole* `self` at once;
  destructuring states the disjointness so the borrow checker accepts it. `&*store`
  reborrows `&mut S` as `&S`, which coerces to `&dyn Store`.

**How do `relate`, `unrelate`, `set_one` differ?**
- `relate` — append a relationship (additive).
- `unrelate` — remove relationships filling `def` (all, or only those pointing at a
  given target); returns the count, notifies only if something changed.
- `set_one` — remove all edges for `def`, then add one. The correct primitive for a
  `one`-cardinality fact: re-stating updates in place instead of appending a duplicate
  that `validate` would flag as `CardinalityError`.

**Trace `closure`: why cycle-safe, why `start` pre-marked seen?**
- DFS: `stack` of nodes to visit, `seen` to avoid revisiting. `seen = HashSet::from([start])`
  pre-marks `start`, so even if a cycle leads back to it, it's never re-emitted (and the
  result *excludes* `start`). Each node is pushed exactly once — on first discovery,
  when `seen.insert(n)` returns `true` — so `order` needs no second dedup pass.

## Rust

- Trait objects (`&dyn Store`, `Box<dyn ChangeSink>`) vs generics: dynamic vs static
  dispatch.
- Destructuring `let Graph { store, sinks } = self;` for per-field borrows; `iter_mut`
  for `&mut` elements.
- `Option::as_ref()` borrows inner without moving; `is_some_and(|t| ...)` true when
  `Some` and the closure holds (`unrelate`'s target match).
- `HashSet::from([start])` builds a set from an array literal.

## Note

`delete` collects affected owners *before* deleting (while the reverse index still lists
them), deletes, then notifies each stripped owner (`notify_put`) plus the removed entity
(`notify_delete`) — so derived artifacts refresh for everything actually touched.
