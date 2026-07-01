# Step 12 — The CLI binary

File: `crates/km-cli/src/main.rs`. A thin binary driving the example end to end.

## Understand

**How does `load_or_die` return a `MemStore` even though the closure calls `exit`?**
- `MemStore::load(path).unwrap_or_else(|e| { eprintln!(...); std::process::exit(1); })`.
- `std::process::exit` returns `!` (the "never" type — it never returns). `!` coerces to
  *any* type, including `MemStore`, so the `unwrap_or_else` closure type-checks even
  though it never actually produces a store. On `Err`, the process just exits.

**Trace one subcommand (`validate`).**
1. `args.get(1).unwrap_or(&default_path)` → snapshot path (`km.db`).
2. `load_or_die(path)` → `MemStore` (or exit).
3. `km_core::validate_all(&s)` → `Vec<Violation>`.
4. Empty → print `ok: 0 violations (N entities)`. Non-empty → print each `{x:?}` to
   stdout, count to stderr, `exit(1)`.

## Rust

- `fn main()` entry point; returns `()` here (errors handled by exiting; `main` could
  instead return `Result`).
- `std::env::args().skip(1).collect::<Vec<String>>()` — drop arg 0 (program name).
- `args.first().map(String::as_str).unwrap_or("help")` — default command.
- `match cmd { "seed" => ..., _ => ... }` on a `&str`; the `_` arm prints usage.
- `eprintln!` → stderr, `println!`/`print!` → stdout.
- Cross-crate `use km_core::{...}` — the other workspace crate, reached via its
  `pub use` front door.

## Subcommand map

| cmd | does |
|-----|------|
| `seed [path]` | seed meta set, save snapshot |
| `load-example [path]` | build full example, save |
| `dump [path]` | `data_view` of every entity |
| `validate [path]` | `validate_all`, print violations, exit 1 if any |
| `schema [path]` | `schema_view` |
| `view <id> [path]` | `data_view` of one entity |
| `neighbors <id> [path]` | `neighborhood` of one entity |
| `by-type <set> [path]` | entities of a set (build-on aware) |
