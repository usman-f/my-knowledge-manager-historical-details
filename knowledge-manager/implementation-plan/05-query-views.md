# Query, Predicates & Views

## Read / write / traversal API (`query.rs`)
Clean seam so processing layers (06) wire in without touching the core.
```rust
pub struct Graph<S: Store> { store: S }

impl<S: Store> Graph<S> {
  // write (append) + edit/retract (all route through put -> ChangeSink)
  fn add_entity(&mut self, name: &str) -> Id;
  fn relate(&mut self, owner: Id, def: DefRef, target: Target, q: Option<Literal>);
  fn set_one(&mut self, owner: Id, def: DefRef, target: Target, q: Option<Literal>); // replace-or-insert a one-card fact
  fn unrelate(&mut self, owner: Id, def: DefRef, target: Option<Target>) -> usize;    // retract: target=Some(t) one edge, None all of def
  fn assign_set(&mut self, e: Id, set: Id);            // sugar: relate via meta.set
  fn unassign_set(&mut self, e: Id, set: Id) -> bool;  // sugar: unrelate the meta.set edge
  fn delete(&mut self, id: Id);                        // whole entity + cascade: inbound links stripped from their owners (no dangling)

  // read
  fn entity(&self, id: Id) -> Option<Entity>;
  fn by_name(&self, name: &str) -> Vec<Id>;
  fn rels(&self, e: Id, def: Option<DefRef>) -> Vec<Relationship>;     // outgoing, optional filter
  fn incoming(&self, target: Id, def: Option<DefRef>) -> Vec<Edge>;    // reverse index

  // traversal
  fn neighbors(&self, e: Id, def: DefRef, dir: Dir) -> Vec<Id>;
  fn closure(&self, start: Id, def: DefRef, dir: Dir) -> Vec<Id>;      // transitive, cycle-safe
}
pub enum Dir { Out, In }
```

## Computed predicate layer (`predicate.rs`)
Order/arithmetic are **computed over typed literals**, never stored edges (design decision).
```rust
pub fn cmp(a:&Literal, b:&Literal) -> Option<Ordering>;  // Number/Date ordered; same-type only
pub fn gt/lt/ge/le(a,b) -> bool;
pub fn add/sub(a:&Literal,b:&Literal) -> Option<Literal>; // Number; Date±duration later
```
- Used by query filters (`range`, `>`, `<`) and constraint checks. No `>`/`<` relationship ever materialized.
- Quantity = `(Literal::Number, unit_entity: Id)` pair at the relationship level (value literal + unit entity); predicates compare quantities only when units match (unit conversion deferred — `eu:m3/h converts-to` is data the store holds, applied by a processor).

## Views (`view.rs`)
Derived projections; cache hot ones, invalidate on `put` of touched ids.
| View | Definition | Built from |
|------|-----------|------------|
| schema | sets + their definitions, qualified names | `SetView`/`DefView`, `meta.defines` |
| data | entity with effective sets resolved + rels grouped by def | `effective_sets` + `defs_for` |
| tree | follow `contains` (and `located-in`) | `closure(.., Dir::Out)` |
| by-type | entities whose effective sets ∋ X | `set_members` + reverse build-on |
| neighborhood | an entity's rels, filtered by def/qualifier, both directions | outgoing + `incoming` |

```rust
pub enum View { Schema, Data(Id), Tree{root:Id, via:DefRef}, ByType(Id), Neighborhood{e:Id, filter:RelFilter} }
pub fn render(graph, View) -> ViewResult;   // serializable; CLI/JSON/embedding input
```
- `by-type` must include entities reaching X through build-on (a `Transmitter` answers a `by-type Instrument` query) — uses `effective_sets`, not just direct `set_members`.
- `neighborhood` surfaces incoming edges with qualifier (e.g. `02FIC100` shows incoming `Data flow "PV"` from `02FIT100`) via reverse index.

## CLI (`km-cli`)
Thin commands over `Graph`: `seed`, `load <file>`, `dump`, `validate`, `view <kind> <id>`, `query`. Output JSON. Drives the example.md dataset end-to-end as the integration test.
