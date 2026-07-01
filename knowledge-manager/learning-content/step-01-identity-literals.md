# Step 1 — Identity & primitive values

Files: `src/id.rs`, `src/literal.rs`.

## Understand

**Why `Id(pub u64)` instead of a raw `u64`?**
- Newtype pattern: a `u64` wearing a distinct type. The compiler rejects mixing an
  `Id` with a plain number, a count, or any other `u64`-shaped value — bugs caught at
  compile time, not runtime.
- Zero cost: identical bits/layout to `u64`; the wrapper exists only for the type
  checker.
- Lets `Id` carry its own trait impls (`Display` prints `#42`) without affecting other
  numbers.

**Why does `as_str` return `Option<&str>` but `as_number` returns `Option<f64>`?**
- A `String` owns heap data. Returning `&str` *borrows* it — no allocation, lifetime
  tied to `&self`. Returning an owned `String` would force a clone on every read.
- `f64` is `Copy` (8 bytes). `*n` copies it out trivially; there's nothing to borrow,
  so an owned value is both simpler and free.
- Rule: borrow what's expensive to own (String), copy what's cheap (number/bool).

**What does `impl From<&str> for Literal` give for free?**
- The reverse: `"x".into()` produces a `Literal` (every `From<A> for B` auto-provides
  `A: Into<B>`).
- Anywhere an `impl Into<Literal>` / `impl Into<Target>` argument is accepted, you can
  pass a bare `&str`. This is why the seed/example code writes `r(DEF, "def")` instead
  of `Literal::String("def".into())`.

**How can `IdGen::alloc` take `&self` and still hand out new ids?**
- `next: AtomicU64`. Atomics offer *interior mutability*: `fetch_add` mutates the value
  through a shared `&` reference (thread-safely, no lock).
- So an `IdGen` only needs to be shared (`&self`), not uniquely borrowed (`&mut self`),
  to allocate — multiple holders can allocate concurrently.
- `fetch_add(1, Relaxed)` returns the value *before* the add (so the first id is
  `USER_ID_START`); `Relaxed` is the weakest ordering, fine because only uniqueness
  matters here, not ordering relative to other memory.

## Rust

- `#[derive(...)]` auto-implements traits: `Copy/Clone` (cheap duplication), `Eq/Hash`
  (map keys), `Ord` (sort/compare), `Debug` (`{:?}`), `Serialize/Deserialize` (serde).
- Tuple struct `Id(pub u64)`; field accessed as `.0`.
- `enum Literal` is a tagged union: exactly one variant at a time, each carrying its own
  payload (`Number(f64)`, `String(String)`, …).
- `match` is exhaustive — add a 5th `Literal` variant and `lit_type`/`as_*`/`Display`
  all fail to compile until handled. `_ =>` is the catch-all arm.
- `Option<T>` = `Some(x)` | `None`; Rust's null-free "maybe".
- `Display`/`write!` define user-facing formatting; `const` is a compile-time constant.
- `as` is an explicit primitive cast (`i64 as f64`); Rust never converts numbers
  implicitly.

## Try it — `Literal::as_date`

Add to `impl Literal` in `literal.rs`:

```rust
pub fn as_date(&self) -> Option<Date> {
    match self {
        Literal::Date(d) => Some(*d), // Date is Copy, so *d copies it out
        _ => None,
    }
}
```

`Date` derives `Copy` (line 27), so this mirrors `as_number`/`as_bool` (copy out), not
`as_str` (borrow).
