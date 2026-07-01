# Step 4 — The storage trait & in-memory backend

File: `src/store/mod.rs`. Read `borrow-checker-guide.md` right after this.

## Understand

**Single source of truth vs cache?**
- Truth: the `entities: HashMap<Id, Entity>` table.
- Cache: `idx: Indexes` (by-name, set-members, reverse, by-def). Always rebuildable from
  the entities alone — `rebuild_indexes` drops and regenerates it. Every mutation routes
  through `put`/`delete` so the cache never drifts.

**In `put`, why `.cloned()` the old entity before unindexing?**
- To unindex the *previous* value at this id you must read it, then overwrite with the
  new `e`. `.cloned()` takes an owned `Option<Entity>` so the borrow of `self.entities`
  ends, leaving `self.idx` free to mutate and the map free to `insert`.
- (Reading `self.entities` while writing `self.idx` is already legal — disjoint fields.
  The owned copy matters more in `delete`, where the value is both mutated and moved.)

**`delete` + `inbound_owners`: how is "no dangling links" enforced? Why strip first?**
- Invariant: no relationship may target a missing entity. Before removing `id`, every
  owner that points at it (named by the reverse index via `inbound_owners`) has those
  relationships stripped (`retain(|r| r.target.entity() != Some(id))`), re-`put` so its
  indexes update.
- Stripping inbound links is what *upholds* the invariant; `dangling_targets` should
  therefore always return empty. Both endpoints stay consistent because `put` maintains
  the reverse index in lockstep.

**What does `+ ?Sized` enable on `dangling_targets`?**
- It relaxes the implicit `Sized` bound, so `store: &S` also accepts an unsized trait
  object `&dyn Store` (whose size isn't known at compile time), not just a concrete
  sized type.

## Rust

- `trait Store` = interface; backends supply bodies in `impl Store for ...`. `&self`
  reads, `&mut self` mutates. Generic functions are `<S: Store>`.
- HashMap entry API: `.entry(k).or_default().push(v)` = get-or-create-then-append.
- `.retain(|x| ...)` keeps matching elements; `.cloned()` turns `Option<&T>` →
  `Option<T>`; `sort` + `dedup` (dedup only collapses *adjacent* equals).
- `let Some(e) = ... else { continue }` (let-else): bind on success, diverge otherwise.
- Iterator pipeline `.iter().map(...).collect()` with the binding's type annotation
  steering `collect`.

## Try it — `count_by_def`

Add to `impl MemStore` (it can touch the private `entities` field):

```rust
/// Total relationships across the store that fill `def`.
pub fn count_by_def(&self, def: Id) -> usize {
    self.entities
        .values()
        .map(|e| e.rels.iter().filter(|r| r.def() == def).count())
        .sum()
}
```

A single lazy pass per entity, summed; nothing allocated.
