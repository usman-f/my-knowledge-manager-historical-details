# First-Change Walkthrough

A guided exercise that threads one new feature through the layers, so you see how a
change ripples through the architecture. Do it after Step 8 of `code-base-learning.md`
(you'll touch schema, validation, views, and tests). Every code block below is
verified to compile and pass.

## Which layers a change touches

| Kind of change | bootstrap | schema decode | validate | view | example | test |
|---|---|---|---|---|---|---|
| New **constraint kind** (Walkthrough 1) | — | yes | yes | auto | optional | yes |
| New **definition field** via a new meta def (Walkthrough 2) | yes | yes | maybe | yes | optional | yes |

Not every change hits every layer. The two walkthroughs together cover them all.

---

## Walkthrough 1 — add a `min-length` string constraint

Goal: let a definition declare `min-length:N`, and have validation reject string
values shorter than `N`. Constraints already exist (`range:`, `allowed:`, `set:`), so
no new meta def is needed — this rides the existing `meta.constraint` plumbing.

### Step 1 — extend the `Constraint` enum (`src/schema.rs`)

Add a variant. The moment you do, the compiler will force you to handle it everywhere
a `Constraint` is matched — that's the type system steering your edit.

```rust
pub enum Constraint {
    Range { min: f64, max: f64 },
    Allowed(Vec<Literal>),
    AllowedSet(Id),
    MinLength(usize),   // <-- new
}
```

### Step 2 — parse it (`src/schema.rs`, `parse_constraint`)

Add a branch before the final `None`. Mirrors the existing `strip_prefix` parsers.

```rust
    if let Some(rest) = raw.strip_prefix("min-length:") {
        return rest.trim().parse::<usize>().ok().map(Constraint::MinLength);
    }
    None
}
```

Rust notes: `strip_prefix` → `Option<&str>`; `parse::<usize>()` turbofish names the
target type; `.ok()` drops the parse error into an `Option`; `.map(Constraint::MinLength)`
passes the enum variant as a *constructor function*.

### Step 3 — enforce it (`src/validate.rs`, `literal_satisfies`)

The `match` over `Constraint` is now non-exhaustive — it won't compile until you add
the arm. (Try building before you add it: the error names the missing variant. This
is why enums + exhaustive `match` make refactors safe.)

```rust
        Constraint::AllowedSet(_) => false,
        Constraint::MinLength(min) => match l.as_str() {
            Some(s) => s.chars().count() >= *min,   // count chars, not bytes
            None => false,                            // non-string -> fails
        },
    }
}
```

Note: `literal_satisfies` is already called for both the target *and* the qualifier
(`validate_entity`), so your constraint guards both for free.

### Step 4 — views: nothing to do

`schema_view` (`src/view.rs`) prints constraints with the `{c:?}` Debug formatter, and
your variant derives `Debug` via the enum's `#[derive(... Debug ...)]`. It renders as
`MinLength(3)` automatically. (To customize the wording you'd format it explicitly.)

### Step 5 — prove it with a dedicated test

Don't bolt this onto the big `example::build` dataset (you'd have to update existing
assertions). Write an isolated test that seeds the meta core, declares one def, and
checks both the happy and failing paths. Create `crates/km-core/tests/min_length.rs`:

```rust
use km_core::bootstrap::{
    seed, DEF_META_CARDINALITY, DEF_META_CONSTRAINT, DEF_META_DEFINES, DEF_META_KIND,
    DEF_META_TARGET_TYPE, SET_META,
};
use km_core::model::{Entity, Relationship, Target};
use km_core::schema::{def_view, Constraint};
use km_core::{validate_entity, MemStore, Store, Violation};

fn r(def: km_core::Id, t: impl Into<Target>) -> Relationship {
    Relationship::new(def, t)
}

#[test]
fn min_length_constraint_decodes_and_validates() {
    let mut s = MemStore::new();
    seed(&mut s);

    // A string def carrying min-length:3.
    let d_note = s.alloc_id();
    s.put(Entity::with_rels(
        d_note,
        "note",
        vec![
            r(DEF_META_KIND, "def"),
            r(DEF_META_TARGET_TYPE, "string"),
            r(DEF_META_CARDINALITY, "one"),
            r(DEF_META_CONSTRAINT, "min-length:3"),
        ],
    ));
    // Hang it on the universal meta set so every entity may use it.
    let mut meta = s.get(SET_META).unwrap();
    meta.rels.push(r(DEF_META_DEFINES, Target::Entity(d_note)));
    s.put(meta);

    // Decoding picks up the new constraint.
    assert_eq!(def_view(&s, d_note).unwrap().constraint, Some(Constraint::MinLength(3)));

    // Valid: length >= 3.
    let ok = s.alloc_id();
    s.put(Entity::with_rels(ok, "ok", vec![r(d_note, "abc")]));
    assert!(validate_entity(&s, ok).is_empty());

    // Invalid: length < 3 -> one ConstraintViolation.
    let bad = s.alloc_id();
    s.put(Entity::with_rels(bad, "bad", vec![r(d_note, "ab")]));
    assert_eq!(
        validate_entity(&s, bad),
        vec![Violation::ConstraintViolation { owner: bad, def: d_note }],
    );
}
```

### Step 6 — run it

```
cargo test --test min_length        # just this file
cargo test --all-features           # confirm nothing else regressed
```

### What you exercised

- Adding an enum variant and letting the compiler find every `match` to update.
- The decode path (`meta.constraint` string → typed `Constraint`).
- The validation path (target + qualifier checked against the constraint).
- Writing a focused integration test instead of mutating shared fixtures.

---

## Walkthrough 2 — add a `description` field to definitions

Goal: definitions carry a human-readable `description`. Unlike a constraint, this is a
*new kind of fact about a definition*, so it needs a **new meta def** — the signature
"schema is itself data" ripple. (It isn't validated, so this one skips `validate.rs`.)

### Step 1 — reserve the meta def (`src/bootstrap.rs`)

```rust
pub const DEF_META_DESCRIPTION: Id = Id(11);   // new, after DEF_META_TARGET (10)
```
Then bump the guard so the validator still treats it as bootstrap:
```rust
const MAX_BOOTSTRAP_ID: u64 = 11;
```
(`USER_ID_START` is 1024, so there's ample room below it.)

### Step 2 — seed it and attach it to the meta set (`src/bootstrap.rs`, `seed`)

```rust
    store.put(meta_def(DEF_META_DESCRIPTION, "description", "string", "one", None));
```
and add a `defines` edge inside the `meta` entity's `rels` list:
```rust
            r(DEF_META_DEFINES, DEF_META_DESCRIPTION),
```

### Step 3 — decode it (`src/schema.rs`)

Add the field to `DefView`:
```rust
    pub description: Option<String>,
```
The struct literal in `def_view` now won't compile until you populate it — the
compiler points you at the exact spot. Add:
```rust
    let description = first_string(store, def_id, DEF_META_DESCRIPTION);
    Some(DefView {
        // ...existing fields...
        description,
    })
```

### Step 4 — surface it in a view (`src/view.rs`, `schema_view`)

Inside the `for d in &sv.defines` loop, after the existing `def` line:
```rust
                if let Some(desc) = &dv.description {
                    out.push_str(&format!("    -- {desc}\n"));
                }
```

### Step 5 — set and read it

In a test (or `example.rs`), give a def a description by adding
`r(DEF_META_DESCRIPTION, "the controller's setpoint")` to its rels, then assert
`def_view(...).unwrap().description.as_deref() == Some("the controller's setpoint")`.
Run `cargo run -p km-cli -- load-example km.db && cargo run -p km-cli -- schema km.db`
to see it rendered.

### What you exercised

- The full "schema is data" loop: a new field of a definition is just **another
  entity (`description`) defined by the meta set**, plus a `meta.defines` edge.
- Compiler-driven editing: a new struct field breaks `def_view` until handled; a new
  bootstrap id needs `MAX_BOOTSTRAP_ID` bumped or the validator would reject the meta
  def as a normal entity with an undefined relationship.

---

## General lesson

A change ripples **down the decode chain**: raw `meta.*` edges (bootstrap) → typed
view (schema) → rules (validate) → rendering (view) → data (example) → proof (test).
The compiler enforces the middle of that chain for you — exhaustive `match` and
all-fields struct literals mean you physically cannot forget a spot. Lean on it: make
the change, build, and let the errors be your checklist.
