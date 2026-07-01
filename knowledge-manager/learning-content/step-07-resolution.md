# Step 7 — Set resolution (effective sets & available defs)

File: `src/resolve.rs`.

## Understand

**Why are "set assignment" and "build-on" the *same* `meta.set` edge?**
- An entity is assigned a set via `meta.set`. A set builds on another set via `meta.set`
  on the *set* entity. Same def (`DEF_META_SET`), so `effective_sets` follows it
  uniformly from both data entities and set entities — composition, not inheritance.
- `direct_sets(store, x)` returns the `meta.set` targets of *any* entity `x`, used both
  for the starting entity and for each parent during the walk.

**How does `effective_sets` avoid infinite loops on a cycle?**
- Iterative graph walk with an explicit `stack: Vec<Id>` and a `seen: HashSet<Id>`.
- `if !seen.insert(set) { continue; }` — `insert` returns `false` if already present, so
  a set is expanded at most once. A cycle (A builds-on B builds-on A) terminates because
  the second visit is skipped. No recursion → no stack-overflow risk.

**Why is the universal `meta` set always included?**
- `out.insert(SET_META)` unconditionally. The meta defs (`meta.set`, `meta.kind`, …)
  must be available to *every* entity for it to be describable/validatable. The meta set
  applies to all.

## Rust

- `BTreeSet` (sorted → deterministic output) for the result; `HashSet` (faster,
  unordered) purely for cycle tracking.
- `while let Some(set) = stack.pop()` loops until the stack empties (`pop` →
  `Option`).
- `set.insert(x)` returning `bool` = compact, allocation-free dedup/cycle guard.
- `defs_for` dedups by qualified name with `seen_names.insert(key)`; the fallback key is
  built lazily via `.unwrap_or_else(|| format!(...))` — closure so the `String` is only
  allocated when actually needed (vs eager `.unwrap_or(x)`).

## How the two functions relate

- `effective_sets(e)` → the set membership (direct + build-on closure + meta).
- `defs_for(e)` → every def those sets hold via `meta.defines`, deduped by `Set.name`.
  This is the list `validate` checks each relationship against.
