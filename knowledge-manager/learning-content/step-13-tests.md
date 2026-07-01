# Step 13 — Tests (how correctness is pinned down)

Files: `tests/example.rs`, then `tests/persist.rs`.

## Understand

**Why can these tests only see `pub` items of `km-core`?**
- Files under `tests/` are *integration tests*: each compiles as its own crate that
  depends on `km-core` externally. Only the public API (`pub mod` / `pub use` in
  `lib.rs`) is reachable — exactly what a real downstream consumer sees. Private
  helpers/fields aren't visible, which keeps the public surface honest.

**In `persist.rs`, what does the inner `{ }` block accomplish?**
- Scope-bounded drop (RAII). Rust has no GC; a value is freed deterministically when it
  leaves scope. The inner block ends → the `RedbStore` `s` is dropped → the database
  file is closed/flushed. The reopen *after* the block then sees a fully committed,
  released file. Without the scope, the open handle would still hold the file.

**Run `cargo test --all-features`; read one failure.**
- 44 tests across 5 suites in three crates: `example`, `persist` (km-core); `embed`,
  `hnsw` (km-edge-enrich-embed); `llm` (km-edge-enrich-llms). Only `persist` and `hnsw`
  are feature-gated; the rest run on a plain `cargo test`. Break an assert (e.g. change
  an expected name) and the `assert_eq!` prints both sides via `Debug`, so the failure
  tells you exactly what was produced.

## Rust

- `#[test]` fns run under `cargo test`; a test passes unless it panics.
  `assert!`/`assert_eq!`/`assert_ne!` panic on failure; the optional trailing message
  embeds values via `Debug` (`{v:?}`).
- `#![cfg(feature = "persist")]` — *inner* attribute (note `!`) gating the whole file on
  a feature.
- Closures capturing scope: `let names = |e| effective_sets(&s, e)...` borrows `s` from
  the enclosing function — something a plain `fn` can't do.
- `.unwrap()` is acceptable in tests (a panic = a failed test); avoid it on production
  paths.
- Custom `ChangeSink` test doubles (`Recorder`, `Rec`) use `Rc<RefCell<Vec<Id>>>` to
  record callbacks and assert the notify wiring.

## What the suite proves

- `example.rs`: clean validation; effective sets match the design; by-type follows
  build-on; reverse index yields incoming data flow; cycle-safe closure excludes start;
  qualified-name disambiguation; one negative test *per* `Violation` variant; edit/retract
  (`set_one`, `unrelate`, `unassign_set`); delete leaves no dangling links and notifies
  the right sinks; snapshot round-trip.
- `persist.rs`: redb round-trips the example, reopens identically, strips inbound links
  durably, and persists the id counter across reopen.
