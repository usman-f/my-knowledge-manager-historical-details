# Step 14 — Persistence backend (redb) *(advanced)*

File: `src/store/redb.rs` (feature `persist`). Durable, embedded, ACID, single file.

## Understand

**Why keep the *same* in-memory `Indexes` and rebuild on `open`? Why is the entity table
the only durable source of truth?**
- Indexes are derivable from entities. Persisting them too would be redundant and could
  *drift* from the entities (two sources of truth). So redb stores only the entity table
  (+ a `next_id` in the `META` table) and, on `open`, scans every entity and calls
  `idx.index(&e)` to rebuild the *identical* `Indexes` struct `MemStore` uses.
- Result: index reads (`by_name`, `incoming`, `rels_by_def`, `members_of_set`) are
  identical to `MemStore`, and the on-disk format stays minimal and impossible to
  corrupt via stale indexes — mirroring the design's "cache is regenerable" stance one
  level down.

**How each mutation stays consistent.**
- `put_checked`/`delete_checked` write the entity (and bump `next_id`) inside one
  `begin_write()` … `commit()` transaction (ACID), *then* update the in-memory indexes
  (`unindex` old, `index` new). `delete` first strips inbound links (same invariant as
  `MemStore`), each cleaned owner re-`put`.

## Rust

- Functions as `map_err` arguments: `.map_err(io)` / `.map_err(enc)` where `io`/`enc` are
  tiny `fn(E: Display) -> StoreError` — a function name is a value.
- Scope-bounded resources: a bare `{ ... }` block bounds a transaction's lifetime so it
  commits and releases before the next use (RAII / `drop`).
- `.expect("msg")` unwraps `Ok` or panics — deliberate here: the `Store` trait is
  infallible, but redb I/O can fail, and a disk failure is treated as unrecoverable
  rather than threaded through every caller. Use `expect` only when an error genuinely
  shouldn't occur in normal operation; otherwise return a `Result` (as the internal
  `*_checked` methods do).
- `TableDefinition<K, V>` typed tables; `bincode` (de)serializes each `Entity` to bytes.

## Note

`RedbStore` and `MemStore` are interchangeable through the `Store` trait — every generic
function (`seed`, `example::build`, `validate_all`, resolution, views) works on either
unchanged. That's the payoff of programming to the trait in Step 4.
