# Core Model — Rust Types

## Ids (`id.rs`)
```rust
#[derive(Copy, Clone, Eq, PartialEq, Hash, Ord, PartialOrd, Serialize, Deserialize)]
pub struct Id(pub u64);            // opaque, dense, sortable; u64 keeps KV keys small

pub struct IdGen { next: AtomicU64 } // monotonic; persisted as a reserved KV key
```
- `name` is an intrinsic field on the entity, not an id. Human-facing strings (`"02FIT100"`, `"meta"`) are names; lookup-by-name is an index, not identity.
- Well-known bootstrap ids reserved at low numbers (see `03-bootstrap.md`) so seed code is deterministic.

## Literals (`literal.rs`)
```rust
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub enum Literal {            // self-typing: tag carries the type
    Number(f64),              // unifies int/float; ordering via predicate layer
    String(String),
    Bool(bool),
    Date(time::Date),         // calendar date; extend to DateTime if needed
}
#[derive(Copy, Clone, Eq, PartialEq, Serialize, Deserialize)]
pub enum LiteralType { Number, String, Bool, Date }

impl Literal { pub fn lit_type(&self) -> LiteralType; }
```
- Primitive types ARE the host-language primitives (per design bootstrap). `LiteralType` is the schema-side tag used by definitions/constraints.
- Number as `f64`: simplest single numeric type; if exact integers/decimals matter later, widen to `enum { Int(i64), Float(f64) }` — isolated to this file.

## Target & relationship (`model.rs`)
```rust
#[derive(Clone, PartialEq, Serialize, Deserialize)]
pub enum Target {
    Entity(Id),               // a node OR a set entity; level distinguishes (validated)
    Literal(Literal),
}

// Reference to a relationship definition. Identity = qualified `Set.name`.
// Stored as the Id of the def entity; DefRef caches the qualified-name for display.
#[derive(Clone, PartialEq, Serialize, Deserialize)]
pub struct DefRef { pub def: Id }          // resolves to a def entity (meta.kind="def")

#[derive(Clone, PartialEq, Serialize, Deserialize)]
pub struct Relationship {
    pub rel_type: DefRef,         // which definition this instance fills
    pub target: Target,           // entity id or literal
    pub qualifier: Option<Literal>, // self-typed; type/constraint declared on the def
}
```

## Entity (`model.rs`)
```rust
#[derive(Clone, Serialize, Deserialize)]
pub struct Entity {
    pub id: Id,
    pub name: String,             // intrinsic, ergonomic; not identity
    pub rels: Vec<Relationship>,  // ordered, append-mostly; dups allowed (multi-edges)
}
```
- One uniform shape for everything: data nodes, value-entities, sets, definitions, the meta set. No subtype enum — "kind" is expressed by relationships (`meta.kind -> "set" | "def"`), per design.
- `rels` is a `Vec` not a map: design allows **multiple relationships between the same two entities** and `many` cardinality. Grouping/dedup is the index's job, not the storage shape.

## Qualified names
- A definition's identity is `Set.name` (e.g. `Transmitter.output-signal`). Stored structurally: the def entity is held by a set via `meta.defines`, and carries its own `name`. `Set.name` is **derived** by `owning_set.name + "." + def.name`, computed in `schema.rs`, cached in `DefRef` for display/parse.
- Parsing `"Set.name"` → `DefRef`: name-index lookup of the set, then scan its `meta.defines` for a child named `name`. Composed sets never collide because the set scopes the name.

## Why these shapes
- `Vec<Relationship>` + `Vec`-of-bytes entity ⇒ one KV row per entity; cheap whole-entity read/write; indexes derive the rest.
- `Target`/`Literal` enums make the entity-vs-literal boundary a compile-time exhaustive match — the central design distinction is unmissable in code.
- No `Property` type anywhere — folding into `Relationship` is structural, not just naming.
