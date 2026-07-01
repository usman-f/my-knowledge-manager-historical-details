# Storage — Store Trait, Backends, Indexes

Implemented: `MemStore` (HashMap + bincode snapshot) and `RedbStore` (feature `persist`). Delta from the sketch below: redb persists only the `entities` table + an id-counter in a `meta` table; the secondary indexes are **in-memory**, rebuilt by scanning `entities` on open (not stored as redb tables) — same "indexes are regenerable cache" principle, less write amplification.

## Store trait (`store/mod.rs`)
Minimal, backend-agnostic. Entity is the unit of read/write; indexes derived.
```rust
pub trait Store {
    fn get(&self, id: Id) -> Result<Option<Entity>, StoreError>;
    fn put(&mut self, e: Entity) -> Result<(), StoreError>;     // upsert; rebuilds that entity's index entries
    fn delete(&mut self, id: Id) -> Result<(), StoreError>;     // also drops reverse-index entries
    fn alloc_id(&mut self) -> Id;
    fn iter_ids(&self) -> Box<dyn Iterator<Item = Id> + '_>;

    // index reads (see Indexes)
    fn by_name(&self, name: &str) -> Vec<Id>;
    fn members_of_set(&self, set: Id) -> Vec<Id>;               // direct meta.set holders
    fn incoming(&self, target: Id) -> Vec<(Id, DefRef, Option<Literal>)>; // reverse edges
    fn rels_by_def(&self, owner: Id, def: Id) -> Vec<Relationship>;
}

#[derive(thiserror::Error, Debug)]
pub enum StoreError { NotFound(Id), Io(...), Encode(...), Conflict(...) }
```
- Mutations go through `put`/`delete` so index maintenance is centralized (no caller-managed indexes).
- Transactions: `redb` backend wraps a batch of puts in one write txn; expose `fn transaction(&mut self, f: impl FnOnce(&mut Txn))` in Phase 1.

## Backends
### memory.rs (Phase 0, tests)
```rust
pub struct MemStore {
    entities: HashMap<Id, Entity>,
    idx: Indexes,            // rebuilt incrementally on put/delete
    ids: IdGen,
}
```
### redb.rs (Phase 1, `persist`)
Tables:
- `entities: u64 -> bincode(Entity)` — source of truth.
- `name_idx: &str -> bincode(Vec<u64>)` — name → ids.
- `set_members: u64(set) -> bincode(Vec<u64>)` — direct `meta.set` holders.
- `reverse: u64(target) -> bincode(Vec<Edge>)` where `Edge{owner,def,qualifier}`.
- `meta: &str -> bytes` — id counter, schema version.
Indexes are persisted but **regenerable**: a `rebuild_indexes()` scans `entities`. Treat indexes as cache that happens to be durable.

## Indexes (`store/index.rs`)
Derived, kept in lockstep with `entities`:
| Index | Key → Value | Serves |
|-------|-------------|--------|
| name | name → [Id] | lookup, `Set.name` parse |
| set_members | set Id → [Id] | by-type view, resolution seed |
| reverse | target Id → [(owner, def, qualifier)] | incoming edges, neighborhood, Data-flow reverse |
| by_def | (owner, def) → [Relationship] | validation, cardinality checks |
- `meta.set` and `meta.defines` are ordinary relationships, so they fall out of the generic reverse/by_def indexes — no special-casing in storage; only `set_members` is a convenience denormalization of `reverse` filtered to `meta.set`.
- On `put(e)`: diff old vs new `rels`, apply incremental index deltas. On `delete`: **cascade** — first strip inbound links (every owner the `reverse` index lists for this id drops its relationships targeting it, re-`put`), then remove the entity + scrub its own edges from reverse/by_def and its id from name/set_members.
- **Link invariant (referential integrity):** every entity-target relationship points at an entity that exists, so both endpoints know the link — the owner stores the outgoing relationship, the target carries the matching `reverse` edge. Relationships stay directed and stored once (the reverse index is the target's bookkeeping, not a duplicated back-edge). `delete` upholds the invariant; `store::dangling_targets` audits it (should always be empty). Same logic in both backends via the shared `inbound_owners` helper.

## Concurrency
- Phase 0: single-threaded, `&mut self`.
- Phase 1: `redb` gives MVCC reads + serialized writes; wrap store in `RwLock` for the embedded API. No custom locking.

## Why
- One row per entity matches the model's whole-entity unit and keeps writes atomic per node.
- Indexes as regenerable derivations honor "views are derived" and let a corrupted/changed index be rebuilt from `entities` alone.
