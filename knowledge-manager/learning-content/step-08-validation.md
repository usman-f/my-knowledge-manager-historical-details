# Step 8 — Validation (closed defs, open values)

File: `src/validate.rs`. Closed on definitions (every relationship needs an available
def), open at the value level (a missing non-required value is never an error).

## Understand

**Why return `Vec<Violation>` instead of throwing / `panic!`?**
- Errors as ordinary values: collect *all* problems in one pass, inspect/aggregate them
  (the CLI prints each, tests `matches!` on them). Validation failure is expected data,
  not exceptional control flow. A panic would abort on the first issue and lose the rest.

**Trace `check_target`: how are kind, level, set membership, constraint each checked?**
- `match (&dv.target_type, target)`:
  - `(Literal(lt), Literal(l))`: *kind* matches; check `l.lit_type() == lt` (else
    `TargetTypeMismatch`), then *constraint* via `literal_satisfies`.
  - `(EntitySet(allowed), Entity(t))`: *kind* matches; check *level* with guards —
    `Some(Instance) if is_set(t)` or `Some(Schema) if !is_set(t)` → `LevelMismatch`;
    then *set membership* — `if let Some(set) = allowed`, require
    `effective_sets(t).contains(set)` else `OutOfSet`.
  - `_`: kind mismatch (literal where entity expected or vice versa) →
    `TargetTypeMismatch`.
- The qualifier is checked separately in `validate_entity` against the same
  `constraint` ("symmetric with the target").

**Why are set/def entities validated only against meta defs, not domain defs?**
- A set/def is a *schema* node. Its `meta.set` edges are build-on (statements about
  instances), and its own facts are `meta.*`. So when `is_set || is_def`, the available
  defs are just `defines_of(SET_META)` — it's validated as a meta-shaped entity, never
  required to satisfy the instance-level defs its members inherit.

## Rust

- `enum Violation` = a typed list of outcomes; each variant carries the data to report
  it (`owner`, `def`, sometimes `target`/`count`).
- Tuple match `(&dv.target_type, target)` over two enums at once.
- Match guards: `Some(Level::Instance) if is_set(store, *t)`.
- `e.rels.iter().filter(|r| r.def() == dv.id).count()` — single lazy pass for cardinality
  / required.
- `let Some(dv) = ... else { continue }` for the happy-path bind.

## Try it — break the example

```
cargo run -p km-cli -- load-example km.db
cargo run -p km-cli -- validate km.db      # ok: 0 violations
```

Then corrupt and re-validate. The integration test `tests/example.rs` already encodes
one per variant; e.g. set `Controller.setpoint` to `999` (range is `0..100`) →
`ConstraintViolation`; drop the required `Instrument.located-in` → `MissingRequired`;
point `located-in` at the `Unit` *set* → `LevelMismatch`; at `PID` (an Algorithm value)
→ `OutOfSet`.
