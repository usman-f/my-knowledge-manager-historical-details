# Codebase Learning Plan

Ordered path through the workspace (`km-core` / `km-cli`, then the
`km-edge-enrich-*` crates) that also teaches Rust. Steps follow conceptual
dependency: each builds only on earlier ones. Do them in order.

## How to use this

- Track progress with checkboxes: `- [ ]` = not done, `- [x]` = done. Check the
  **Read**, each **Understand before proceeding** checkpoint, and **Try it** as you
  finish them; a step is complete when all its boxes are checked. The **Progress**
  list below mirrors the steps — tick a step there once its boxes are all done.
- Inline comments tagged `// Rust:` explain language features at the point they
  appear. List every lesson: `grep -rn "// Rust:" crates/`.
- "Book" = *The Rust Programming Language* (https://doc.rust-lang.org/book). Chapter
  refs are where to read the concept formally.
- Design rationale lives in `design/*.md`; this plan points to the relevant section
  per step. Code is the *how*; design docs are the *why*.
- Companion docs: `glossary.md` (model vocabulary), `borrow-checker-guide.md` (the #1
  Rust-beginner hurdle, with this codebase's patterns — read after Step 4),
  `first-change-walkthrough.md` (a guided change threaded through the layers — do
  after Step 8). `learning-content/step-NN-*.md` holds the per-step *answers* to the
  "Understand before proceeding" questions, in-context Rust explanation, and worked
  exercise solutions — attempt each step's questions yourself first, then check there.

## Progress

- [ ] Setup (do once)
- [ ] Mental model
- [ ] Step 1 — Identity & primitive values
- [ ] Step 2 — The data model (entities & relationships)
- [ ] Step 3 — Computed predicates over literals
- [ ] Step 4 — The storage trait & in-memory backend
- [ ] `borrow-checker-guide.md` (read after Step 4)
- [ ] Step 5 — Bootstrap: the self-defining meta schema
- [ ] Step 6 — Decoding schema from raw entities
- [ ] Step 7 — Set resolution (effective sets & available defs)
- [ ] Step 8 — Validation (closed defs, open values)
- [ ] `first-change-walkthrough.md` (do after Step 8)
- [ ] Step 9 — Query/Graph API & change notifications
- [ ] Step 10 — Derived views (rendering)
- [ ] Step 11 — The worked example
- [ ] Step 12 — The CLI binary
- [ ] Step 13 — Tests
- [ ] Step 14 — Persistence backend (redb) *(advanced)*
- [ ] Step 15 — Embeddings & vector search *(advanced)*
- [ ] Step 16 — LLM processor (extraction + GraphRAG) *(advanced)*

## Setup (do once)

- [ ] `cargo build` — default build (core only; edge crates build but with no
  optional features).
- [ ] `cargo build --all-features` — adds the optional features: `persist` (km-core),
  `hnsw` (km-edge-enrich-embed), `llm-http` (km-edge-enrich-llms). The edge crates
  themselves build by default; there is no longer an `embed` or `llm` feature (they
  were the in-core gates before the edge-crate split, 2026-06-24).
- [ ] `cargo test --all-features` — 44 tests across 5 suites: `example`, `persist`
  (km-core); `embed`, `hnsw` (km-edge-enrich-embed); `llm` (km-edge-enrich-llms). All
  must pass. Plain `cargo test` skips the feature-gated suites (`persist`, `hnsw`).
- [ ] `cargo run -p km-cli -- load-example km.db && cargo run -p km-cli -- dump km.db`
  — build and print the sample dataset.
- [ ] Workspace = four crates: `crates/km-core` (library, Level 0 pure store),
  `crates/km-cli` (binary `km`, core only), and two edge crates over the core —
  `crates/km-edge-enrich-embed` (Level 1: embeddings/vector search) and
  `crates/km-edge-enrich-llms` (Level 2: LLM processor). Edges depend on km-core;
  km-core never depends on an edge (enforced by the crate boundary). Book ch.1, ch.7
  (packages, crates, modules), ch.14 §workspaces.

## Mental model (read before code)

- [ ] Two building blocks only: **entities** and **relationships**. No separate
  property/class types.
- [ ] Schema (sets, definitions, the meta set) is *itself* entities of the same shape.
- [ ] Order of layers, bottom-up: ids/literals → model → store → bootstrap schema →
  schema decoding → resolution → validation → query/views → example → CLI.
- [ ] Read `design/principles.md` and `design/model.md` §Building blocks now; reread
  per step.
- [ ] Keep `glossary.md` open — a one-page reference for the model's vocabulary
  (entity, set, def, meta set, value-entity, qualifier, level, effective sets, …),
  each term keyed to where it lives in code and design. Look terms up as they appear.

---

## Step 1 — Identity & primitive values

- **Read:**
  - [ ] `src/id.rs`, then `src/literal.rs`.
- **Design:** `design/model.md` §Entity vs literal, §Promoting a value;
  `design/decisions.md` §Keep literals.
- **Rust concepts:** structs (tuple/newtype), `enum` as tagged union, `#[derive]`
  (Copy/Clone/Eq/Hash/Ord/Debug/Serialize), `match` (exhaustive, wildcard, deref),
  `Option<T>`, `Display`/`write!`, `From`/`Into`, `const`, `as` casts, atomics &
  interior mutability (`&self` that still mutates). Book ch.5, ch.6, ch.10.2.
- **Understand before proceeding:**
  - [ ] Why is `Id(pub u64)` a distinct type instead of a raw `u64`? (newtype safety)
  - [ ] Why does `Literal::as_str` return `Option<&str>` but `as_number` returns
    `Option<f64>`? (borrow vs copy)
  - [ ] What does implementing `From<&str> for Literal` give you for free?
  - [ ] How can `IdGen::alloc` take `&self` (not `&mut self`) and still hand out new ids?
- **Try it:**
  - [ ] add a `Literal::as_date(&self) -> Option<Date>` accessor.

## Step 2 — The data model (entities & relationships)

- **Read:**
  - [ ] `src/model.rs`.
- **Design:** `design/model.md` §Relationships, §Building blocks;
  `design/decisions.md` §Drop "properties", §Qualifier.
- **Rust concepts:** generics with trait bounds (`<L: Into<Literal>>`), `impl Trait`
  in argument position, ownership vs borrowing, `&mut self` and method chaining,
  returning `impl Iterator` + `move` closures (laziness). Book ch.4 (ownership),
  ch.10.1 (generics), ch.13 (iterators/closures).
- **Understand before proceeding:**
  - [ ] A `Target` is `Entity(Id)` or `Literal(...)` — why is that distinction "the
    central design boundary"?
  - [ ] Why can `Relationship::new` accept both an `Id` and a `DefRef` for `rel_type`?
  - [ ] What does `rels_of` return, and why does nothing happen until you iterate it?
- **Try it:**
  - [ ] write a free function counting how many relationships an `Entity` has
    for a given `def` (using `rels_of().count()`).

## Step 3 — Computed predicates over literals

- **Read:**
  - [ ] `src/predicate.rs`.
- **Design:** `design/model.md` §Promoting a value ("order/arithmetic are computed,
  not stored").
- **Rust concepts:** the `?` operator on `Option`, `matches!` with or-patterns and
  guards, matching on tuples `(a, b)`, `partial_cmp` vs `cmp` (NaN). Book ch.6.2,
  ch.9 (`?`), ch.18 (patterns).
- **Understand before proceeding:**
  - [ ] Why does `cmp` return `Option<Ordering>` rather than `Ordering`?
  - [ ] Trace `add`: what makes it return `None`? How does `?` short-circuit?
  - [ ] Why is comparison across different literal types `None`, not `false`?

## Step 4 — The storage trait & in-memory backend

- **Read:**
  - [ ] `src/store/mod.rs` (the file that teaches the most Rust idioms).
- **Design:** `design/model.md` §Implementation; module-level doc comment.
- **Rust concepts:** defining a `trait` (interface), `&self` vs `&mut self` methods,
  generic functions `<S: Store>`, `?Sized` and `&dyn`, `HashMap` entry API
  (`.entry().or_default().push()`), iterate-by-reference, `.cloned()` to satisfy the
  borrow checker, `sort`/`dedup`/`retain` with closures, `let ... else`. Book ch.10.2
  (traits), ch.8 (collections), ch.4 (borrowing).
- **Understand before proceeding:**
  - [ ] What is the single source of truth in `MemStore`, and what is "just a cache"?
  - [ ] In `put`, why call `.cloned()` on the old entity before unindexing it?
  - [ ] Read `delete` + `inbound_owners`: how is the "no dangling links" invariant
    enforced? Why strip inbound links *before* removing the entity?
  - [ ] What does `+ ?Sized` enable on `dangling_targets`?
- **Try it:**
  - [ ] add a `MemStore` method `count_by_def(&self, def: Id) -> usize`.

## Step 5 — Bootstrap: the self-defining meta schema

- **Read:**
  - [ ] `src/bootstrap.rs`.
- **Design:** `design/model.md` §Bootstrap, §Sets = the type;
  `design/principles.md` (schema is data).
- **Rust concepts:** generic `&mut S` parameters, `vec!`, `mut` bindings, `if let`,
  module-private helper fns. Reinforces Step 4.
- **Understand before proceeding:**
  - [ ] How is the `meta` set "defined in terms of itself"? Which ids are reserved?
  - [ ] Why does each `DEF_META_*` constant correspond to a field of a definition?
  - [ ] Why must bootstrap ids be skipped by the validator (`is_bootstrap`)?

## Step 6 — Decoding schema from raw entities

- **Read:**
  - [ ] `src/schema.rs`.
- **Design:** `design/model.md` §Sets = the type, §Relationship definitions,
  §Namespacing.
- **Rust concepts:** struct-like vs tuple enum variants, `find_map`, functions passed
  by name (`str::to_string`), `strip_prefix`/`split_once`, `.parse().ok()?`,
  turbofish `parse::<f64>()`, generic `<S: Store + ?Sized>`. Book ch.6, ch.13.
- **Understand before proceeding:**
  - [ ] A `DefView`/`SetView` is never stored — where does its data come from?
  - [ ] Trace `def_view`: how does a raw entity become a typed view via `meta.*` edges?
  - [ ] How does `resolve_def("Set.name")` turn a string into a `DefRef`?

## Step 7 — Set resolution (effective sets & available defs)

- **Read:**
  - [ ] `src/resolve.rs`.
- **Design:** `design/model.md` §Sets = the type ("composition, not inheritance").
- **Rust concepts:** `BTreeSet` vs `HashSet`, `while let Some(x) = stack.pop()`,
  `.insert()` returning `bool` for cycle detection, lazy `unwrap_or_else`. Iterative
  graph walk vs recursion. Book ch.8.3.
- **Understand before proceeding:**
  - [ ] Why are "set assignment" and "build-on" the *same* `meta.set` edge?
  - [ ] How does `effective_sets` avoid infinite loops on a cycle?
  - [ ] Why is the universal `meta` set always included?

## Step 8 — Validation (closed defs, open values)

- **Read:**
  - [ ] `src/validate.rs`.
- **Design:** `design/decisions.md` §Closed definitions, open values;
  `design/model.md` §Relationship definitions.
- **Rust concepts:** enums as a typed list of outcomes (errors as values, not
  exceptions), nested tuple matching `(&target_type, target)`, match guards
  (`Some(Level::Instance) if is_set(...)`), `.filter().count()`. Book ch.9 (error
  handling philosophy), ch.18.
- **Understand before proceeding:**
  - [ ] Why return `Vec<Violation>` instead of throwing/`panic!`?
  - [ ] Trace `check_target`: how are target *kind*, *level*, *set membership*, and
    *constraint* each checked?
  - [ ] Why are set/def entities validated only against meta defs, not domain defs?
- **Try it:**
  - [ ] run `cargo run -p km-cli -- validate km.db`; deliberately break the
    example (e.g. set a value out of range) and watch a `Violation` appear.

## Step 9 — Query/Graph API & change notifications

- **Read:**
  - [ ] `src/query.rs`.
- **Design:** `design/processing-layer.md` §Implication for the store design.
- **Rust concepts:** trait objects (`&dyn Store`, `Box<dyn ChangeSink>`), dynamic vs
  static dispatch, why `Box` is needed for unsized `dyn`, splitting `&mut self` into
  per-field borrows via destructuring, `iter_mut`, `is_some_and`. Book ch.17.2
  (trait objects), ch.10 (generics for contrast).
- **Understand before proceeding:**
  - [ ] `ChangeSink` uses `dyn`; `EmbeddingIndex<E>` uses generics — why each choice?
  - [ ] In `notify_put`, why destructure `self` into `{ store, sinks }` first?
  - [ ] How do `relate`, `unrelate`, `set_one` differ? Which is right for a
    `one`-cardinality fact?
  - [ ] Trace `closure`: how is it cycle-safe and why is `start` pre-marked seen?

## Step 10 — Derived views (rendering)

- **Read:**
  - [ ] `src/view.rs`.
- **Design:** `design/model.md` §Views; `design/principles.md` (views are derived).
- **Rust concepts:** `if` as an expression (no ternary), building owned `String`
  with `push_str`/`format!`, `match` as an expression for unwrap-or-skip. Book ch.8.2.
- **Understand before proceeding:**
  - [ ] Nothing in this file is stored — confirm why (everything recomputed from edges).
  - [ ] Difference between `schema_view`, `data_view`, `neighborhood`, `by_type`.

## Step 11 — The worked example (it all comes together)

- **Read:**
  - [ ] `src/example.rs` alongside `design/example.md`.
- **Rust concepts:** `&[Id]` slice parameters, `_`-prefixed bindings & `let _ =`,
  `#[allow(clippy::...)]`. Reinforces model + store + bootstrap.
- **Understand before proceeding:**
  - [ ] Map each set/def/value-entity in the code to the diagram in `design/example.md`.
  - [ ] Why are `fit`/`fic`/`fy` ids reserved up front (`alloc_id`) before `put`?
  - [ ] Follow one `Data flow` relationship and its qualifier (e.g. "PV") end to end.
- **Try it:**
  - [ ] add a new instrument entity to `build`, re-run `validate`, fix any
    violation it reports.

## Step 12 — The CLI binary

- **Read:**
  - [ ] `crates/km-cli/src/main.rs`.
- **Rust concepts:** `fn main`, `std::env::args`, `match` on `&str`, the `!` (never)
  type via `process::exit`, `eprintln!` vs `println!`, cross-crate `use`. Book ch.12.
- **Understand before proceeding:**
  - [ ] How does `load_or_die` return a `MemStore` even though the closure calls
    `exit`? (the `!` type)
  - [ ] Trace one subcommand from arg parsing to output.

## Step 13 — Tests (how correctness is pinned down)

- **Read:**
  - [ ] `tests/example.rs`, then `tests/persist.rs`.
- **Rust concepts:** integration tests as separate crates (public API only),
  `#![cfg(feature = ...)]` inner attribute, `#[test]`, `assert!`/`assert_eq!`,
  `.unwrap()` in tests, closures capturing scope, RAII drop (`{ }` scope closing a
  DB). Book ch.11.
- **Understand before proceeding:**
  - [ ] Why can these tests only see `pub` items of `km-core`?
  - [ ] In `persist.rs`, what does the inner `{ }` block accomplish (drop = close file)?
  - [ ] Run `cargo test --all-features`; read one failure by breaking an assert.

---

## Advanced track (optional features)

Do these after Steps 1–13. Step 14 is a km-core feature (`persist`); Steps 15–16
move out to the edge crates (`km-edge-enrich-embed`, `km-edge-enrich-llms`), each a
separate crate depending on km-core — read `crates/km-core/src/lib.rs`'s top doc
comment for the layering, and `design/decisions.md` §Edges over a pure core.

### Step 14 — Persistence backend (redb)

- **Read:**
  - [ ] `src/store/redb.rs` (feature `persist`).
- **Rust concepts:** functions used as `map_err` arguments, scope-bounded resources
  (transactions), RAII/`drop`, `.expect()` and *when* panicking is acceptable
  (infallible trait over fallible I/O). Book ch.9.
- **Understand:**
  - [ ] Why does `RedbStore` keep the *same* in-memory `Indexes` as `MemStore` and
    rebuild them on `open`? Why is the entity table the only durable source of truth?

### Step 15 — Embeddings & vector search

- **Read:**
  - [ ] `km-edge-enrich-embed/src/lib.rs`, then `src/hnsw.rs` (feature `hnsw`).
- **Design:** `design/processing-layer.md` §Level 1;
  `design/decisions.md` §Processing is a decoupled layer, §Edges over a pure core.
- **Rust concepts:** lifetime-bound `impl Iterator + '_`, `&mut [f32]` slices,
  multi-method traits, `zip`/`sum`, closures as generic `Fn` params, generic
  `EmbeddingIndex<E>` (static dispatch) implementing the `dyn` `ChangeSink`, lifetimes
  `'static` in `HnswIndex`. Book ch.10.3 (lifetimes), ch.13.
- **Understand:**
  - [ ] Why is the vector a *derived, regenerable* artifact and never a relationship?
    How does `EmbeddingIndex` stay fresh via `ChangeSink`? Why can a real model be
    swapped in by implementing `Embedder` alone?

### Step 16 — LLM processor (extraction + GraphRAG)

- **Read:**
  - [ ] `km-edge-enrich-llms/src/lib.rs` (`llm-http` feature for the real client).
- **Design:** `design/processing-layer.md` §Level 2; `design/decisions.md`
  §Edges over a pure core. The crate's *forward* role is the **enrich** stage
  (gap-fill / mistake-find → claims); the built `populate` (a *load* path) and
  `answer`/`graphrag_context` (a *use* path) are example/legacy — `roadmap/status.md`.
- **Rust concepts:** idiomatic custom error type (`enum` + `Display` +
  `std::error::Error`), serde `#[derive(Deserialize)]` + `#[serde(default)]`, `?`
  with `Result` + `map_err`, feature-gated transports = dependency injection
  (`MockLlm` always present, `AnthropicLlm` only with `llm-http`). Book ch.9, ch.17.
- **Understand:**
  - [ ] Which part is "fuzzy" (the model call) vs "exact and testable" (the
    orchestration)? Trace `populate` (write) and `graphrag_context` (read). Why embed a
    candidate before creating an entity (dedup)?

---

## Rust concept index (where each is taught)

- Ownership / borrowing / lifetimes → Steps 2, 4, 9, 15.
- Structs & enums & pattern matching → Steps 1, 3, 6, 8.
- `Option` / `Result` / `?` / errors-as-values → Steps 1, 3, 8, 14, 16.
- Traits, generics, `impl Trait`, bounds → Steps 2, 4, 6, 15.
- Trait objects (`dyn`, `Box`) vs static dispatch → Steps 9, 15.
- Iterators & closures (`map`/`filter`/`find_map`/`zip`/`fold`) → Steps 2, 6, 15.
- Collections (`Vec`, `HashMap` entry API, `HashSet`, `BTreeSet`) → Steps 4, 7.
- `From`/`Into` conversions → Steps 1, 2.
- Modules, crates, edge crates, `pub use`, features → Setup, `src/lib.rs`, Steps 14–16.
- Testing → Step 13.
- RAII / `drop` → Steps 13, 14.

## Suggested cadence

- One step per session for Steps 1–8 (the conceptual core).
- Steps 9–13 can be paired (9+10, 11+12).
- Advanced track only after the core feels solid.
- After each step: re-run `cargo test`, and `grep "// Rust:"` the files you read to
  review every language note in context.
