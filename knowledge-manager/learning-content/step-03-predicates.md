# Step 3 — Computed predicates over literals

File: `src/predicate.rs`. This is the Level 0 "computed, not stored" layer — order and
arithmetic are functions over literals, never relationships on disk.

## Understand

**Why does `cmp` return `Option<Ordering>` rather than `Ordering`?**
- Two sources of "no answer":
  - Floats include `NaN`, which is unordered — `f64::partial_cmp` returns `None`.
  - Cross-type pairs (number vs string) are meaningless to compare — the `_ => None`
    arm.
- `Option` encodes "incomparable" as a first-class outcome; a plain `Ordering` could
  not.

**Trace `add`: what makes it return `None`? How does `?` short-circuit?**
- `add` = `Some(Literal::Number(a.as_number()? + b.as_number()?))`.
- If either operand isn't a `Number`, `as_number()` is `None`, and `?` *returns `None`
  from the whole function immediately*. Otherwise `?` unwraps the inner `f64`.
- Reads as "add the two numbers, or give up if either isn't numeric" — no nested
  `match`.

**Why is comparison across different literal types `None`, not `false`?**
- Open-world semantics: incomparable ≠ false. `false` would assert a definite
  relationship ("not greater"); `None` says "the question doesn't apply." Returning
  `false` would let nonsense like `number > string` silently succeed as a real ordering.

## Rust

- `match (a, b)` matches a tuple of two values at once; each arm requires both sides to
  be the same variant.
- `partial_cmp` (floats, may be `None`) vs `cmp` (total order — strings/bools/dates,
  always defined, wrapped in `Some`).
- `matches!(value, PATTERN)` — concise bool test. Or-patterns: `Some(Greater | Equal)`.
- Match guard: `matches!(x.as_number(), Some(n) if n >= min && n <= max)` adds a runtime
  condition to a pattern.
- `?` on `Option` propagates `None`; on `Result` propagates `Err`.

## Note

The boolean wrappers (`eq`/`gt`/`lt`/`ge`/`le`) are all defined in terms of `cmp`, so
the NaN/cross-type handling lives in exactly one place.
