# Borrow-Checker Survival Guide

The #1 thing that stops Rust beginners. This is the practical mental model plus the
exact patterns this codebase uses, keyed to real functions. Companion to
`code-base-learning.md` (read after Step 4).

## The three rules

1. Any number of shared borrows `&T`, **or** exactly one mutable borrow `&mut T` —
   never both at once, for the same data.
2. A borrow lasts only until its **last use** (non-lexical lifetimes / NLL), not
   until the end of the block. Stop using it and it's released.
3. You cannot **move** out of, or **mutate** through, a shared borrow `&T`.

Everything below follows from these.

## What is actually allowed (don't over-clone)

- **Disjoint field borrows.** Different fields of the same struct can be borrowed
  independently — even one `&` and one `&mut` at the same time.
  - `MemStore::put` reads `self.entities` (via `.get`) while mutating `self.idx`
    (via `unindex`). No conflict: distinct fields.
  - `Graph::notify_put` mutably iterates `sinks` while passing `&store` to each
    sink. Distinct fields, so this is fine; the code *destructures* `self` mainly to
    state that disjointness plainly.
- **A borrow that ends before the next use.** Because of rule 2, you can read
  through a borrow, finish with it, then take `&mut self` on the next line.

So before reaching for `.clone()`, ask: *are these actually the same data, and is the
borrow still live?* Often the answer is no.

## When you genuinely need to act — the patterns this codebase uses

### Pattern A — Clone to own, when you must mutate or move

If you only have `&T` but need to change the value or move it into a `&mut self`
method, take an owned copy.

- `MemStore::delete`: `self.entities.get(&owner).cloned()` yields an owned `Entity`,
  so the loop can mutate `e.rels` and then move `e` into `self.put(e)`. A `&Entity`
  from the map could do neither (rule 3).

### Pattern B — Collect first, then mutate

Reading from `self` produces a borrow; finish that read into an owned `Vec` so the
borrow ends, *then* mutate `self`.

- `inbound_owners` returns an owned `Vec<Id>`; `MemStore::delete` loops over it and
  calls the `&mut self` method `put` inside the loop — safe because the reverse-index
  borrow was already released into the Vec.
- `Graph::delete`: `let mut affected: Vec<Id> = self.store.incoming(id)...collect();`
  ends the `incoming` borrow before `self.store.delete(id)` runs.

### Pattern C — Destructure `self` for explicit disjoint borrows

When you need two field borrows at once and want it unambiguous (or the compiler
can't see through a whole-`self` method call), bind the fields by name.

- `Graph::notify_put`: `let Graph { store, sinks } = self;` then iterate `sinks`
  mutably while reading `store`.

### Pattern D — `let ... else` to drop the `Option` borrow early

`let Some(e) = store.get(id) else { continue };` unwraps on the happy path and
diverges otherwise — no nested block holding a borrow open.

- Used in `dangling_targets`, `validate_entity`, `Graph::unrelate`, and others.

### Pattern E — Borrow, don't move, when reading

- `for r in &e.rels` iterates by reference (the Vec is still usable afterward).
  `for r in e.rels` would consume it.
- `Option::as_ref()` turns `&Option<T>` into `Option<&T>` so you inspect without
  moving the inner value — see `Graph::unrelate`'s `target.as_ref()`.
- For `Copy` types (like `Id`), `*x` copies the value out of a reference instead of
  moving — see the `match` arms in `model.rs` / `literal.rs`.

## Reading the two errors you'll hit most

### E0502 — "cannot borrow `x` as mutable because it is also borrowed as immutable"

You used a shared borrow *after* a mutating operation on the same data.

```rust
if let Some(v) = self.m.get(&k) {  // immutable borrow of self.m, held...
    self.put(k, String::new());    // ...&mut self here -> CONFLICT
    println!("{v}");               // because v is still used after
}
```
Fix: stop using the borrow first — clone what you need before the mutation
(Pattern A), or restructure so the read fully precedes the write (Pattern B).

### E0499 — "cannot borrow `*self` as mutable more than once at a time"

You took a `&mut` to the whole `self` (or the same field) while another `&mut` to it
was live.

```rust
for s in self.sinks.iter_mut() {   // &mut self.sinks held across the loop
    helper(self);                  // &mut whole self -> CONFLICT
}
```
Fix: borrow only the distinct fields you need (Pattern C), or collect the data you'll
mutate into a Vec first (Pattern B).

## Decision flow when the compiler rejects your code

1. Is it really the *same* data, or different fields? Different fields → borrow them
   separately (no clone needed).
2. Is the borrow still alive where it conflicts? Move its last use earlier, or use
   `let-else` (Pattern D) so it ends sooner.
3. Do I need to mutate/move what I only borrowed? `.clone()` / `.cloned()` to own it
   (Pattern A).
4. Am I reading from `self` then mutating `self`? Collect the read into an owned
   value first (Pattern B).
5. Two field borrows at once? Destructure `self` (Pattern C).

## Cost note

`.clone()` copies data (a `Vec`/`String` clone allocates). It's the right tool when
ownership is genuinely needed, but don't scatter it to silence the compiler before
checking rules 1–3 — often a reference or an earlier-ending borrow is free. `Id`,
`DefRef`, and the like are `Copy`, so copying them is trivial.
