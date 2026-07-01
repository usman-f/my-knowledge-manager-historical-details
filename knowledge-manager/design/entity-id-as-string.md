# Entity Id as string / natural key?

Question: make `Id` a string unique identifier shaped `[database][column][value]` (e.g. `SAP.Main WorkCtr.1234`), so loading can designate columns whose cells become **entities** addressable by a text path; possibly make `name` that path.

## Verdict
- **No** — do not change `Id`. Keep `Id(u64)`: opaque, surrogate, stable.
- The need (stable identity + fast addressing for column-derived entities) is **already met** by existing machinery; the only real gap is a *fast path lookup*, which is an **additive index**, not an id-type change.
- Do **not** make a `[db][col][value]` path the `name` of value-entities — it breaks cross-source dedup.

## Two needs the proposal conflates
1. **Identity** — what an entity *is*, stable for the entity's lifetime, embedded in every inbound edge.
2. **Addressing** — a human/loader-facing handle to *find* an entity quickly.
The proposal overloads (1) with (2). Standard surrogate-vs-natural-key rule: natural keys make poor primary keys (they mutate, aren't universal, collide). Keep them separate.

## Why `Id` stays `u64`
- **Copy/dense/sortable.** `Id(u64)` is `Copy` (`id.rs`). String is not → pervasive `.clone()`/borrow churn through `model.rs`, `store/`, `resolve.rs`, `schema.rs`, `predicate.rs`, every `<S: Store>`.
- **Storage keys.** redb key is `TableDefinition<u64, ...>` (`store/redb.rs`); all in-memory indexes key on `Id` (`reverse`, `by_def`, `set_members`, `by_value`). Every `Target::Entity(Id)` is serialized per edge — a string path per edge bloats a 240-column / millions-of-edges load massively.
- **Monotonic alloc + reserved range.** `IdGen` (`id.rs`) hands out dense ids; `USER_ID_START=1024`; bootstrap is ids `1..=10` via `is_bootstrap` (`bootstrap.rs`). String ids make both meaningless. Contiguous row ids are relied on (`LoadReport.first_row_id`: "ids are contiguous from here").
- **Identity must be stable.** `[db][col][value]` is not: a corrected cell value, a renamed column, or a recategorized row would change the id → rewrite every inbound edge. `delete`/link-invariant logic (`store/mod.rs inbound_owners`) assumes a fixed target id.
- **Not universal.** Many entities have no natural path: keyless rows (already fall back to `{set}#{id}`), the meta set, promoted concept-entities, reified relationships. A mixed scheme (path where available, synthetic otherwise) gains nothing over a uniform surrogate.
- **Not actually unique for rows.** Cell values duplicate within a column (the very reason `name` is non-unique and `by_name -> Vec<Id>`). `[db][col][value]` is unique only for *interned value-entities* (one per distinct value per target set), not for data rows.

## The use case is already built
"Designate columns whose cells become entities, deduped" = **`km-edge-promote`** (`promote` / `promote_all`):
- Interns each distinct literal value once **per target set**, by entity name (`HashMap<String,Id>` seeded from `members_of_set`), links owner→entity, keeps the literal. Cross-source/repeat calls converge (`second_call_dedups_against_existing_members`).
- The loader already does the same for the `source` entity (interned in the `sources` set, `km-edge-load-excel`).
- App decides *which* columns and *which named set* (`Promotion`); km-core/edges hold no domain vocab (decisions: edges over a pure core; consolidator scope rule).
So dedup + stable identity for column-derived entities exists today. The `[db][col]` part of the path ≈ the **target set** the value is interned into; `[value]` ≈ the value-entity name.

## Addressing already partly exists
- Qualified `Set.name` resolves defs: `resolve_def` splits on first `.` → `set_by_name` → match `defines` by name (`schema.rs`). `.` is reserved; loader rewrites `.`→`_` in set names (`set_name`).
- Source sets are named by database (`SAP`, `P&ID/sheet`) — the `[database]` of the path.
- Missing: a single O(1) **entity** lookup by a unique path. `by_name` is ambiguous (`Vec<Id>`); no `by_key`.

## Why not make the path the `name`
- Settled: `name` is the display/lookup handle, identity is `Id` (`name-typing-and-source-provenance.md` Q1). `by_name`/`set_by_name`/display assume a plain string; `.` collides with the `Set.name` separator.
- **Value-entities:** naming them by qualified `db.col.value` would prevent the convergence that is the entire point — `"8"` from `SAP.unit` and `"8"` from `P&ID.unit` must become **one** `Unit` entity. A path-name forks them per source/column. Keep value-entity name = bare value.
- **Row entities:** a readable `db/key` *display* name is fine, but it is non-unique and must not be the identity.

## Recommended (additive) shape if fast path lookup is wanted
Add a generic **natural-key index** over the existing store — same shape as `by_name`/`by_value`, no `Id` change:
- km-core: a `key` index `HashMap<String,Id>` populated from a designated string def per set (or a reserved `meta.key`); `Store::by_key(&str) -> Option<Id>` (unique, unlike `by_name`); `ensure_keyed(key) -> Id` get-or-create for upsert-on-load.
- Key string is built by the **app/edge** (consolidator composes `[database][column][value]`), never hardcoded in km-core (scope rule).
- Benefits: O(1) addressing, idempotent re-load (upsert by key), explicit uniqueness — all while `Id` stays opaque/stable and edges stay cheap.

## Inter-relation does not need a string Id (the core worry)
- "Rows with the same value should link to a shared node" = promotion, already built and **planned for this exact column**: consolidator `docs/promotion-field-review.md` table — `Main WorkCtr` → link `main-workcenter` → set `WorkCenter`.
- Graph-ness comes from the shared **entity target**, not the id's spelling. A `u64`-id `WorkCenter` node is still a node; rows reverse-query off it one hop. A bare literal `"8"` on two rows does not unify them — only a shared entity does (`promotion-field-review.md` §"Why an entity at all"). This is exactly what distinguishes it from a flat SQL column, and it is id-representation-agnostic.
- **Dedup scope = choice of `Promotion.target_set`** (the one real lever):
  - shared set (`WorkCenter`) → converge a value across databases (what the existing six unit/var/role/… promotions do — cross-source SAP↔P&ID↔IE).
  - database-scoped set (`SAP/WorkCenter`) → keep each database's population distinct even on a matching value string (intra-source dedup; `promotion-field-review.md` §"Key difference" — SAP-only dimensions like `Main WorkCtr`/`Cost Center`/`Functional Loc.`).
- The desired path `[database][column][value]` maps onto **set name + entity name**: `[db][col]` = the target set's identity, `[value]` = the entity's name in it. Address it via `by_key`; `Id` stays an opaque surrogate.
- Recipe (no core change): `ensure_set` + `ensure_link_def` + one `Promotion` in `Vocab::promotions()` (`promotion-field-review.md` §"To implement").

## If a string id is still wanted later
Only viable as a *second* stable surrogate (e.g. a UUID/ULID minted once and never derived from mutable fields) — not a `[db][col][value]` path. That buys global uniqueness across stores at the cost of the points above; revisit only if multi-store merge needs it. A derived-from-content id is ruled out for the mutability/collision reasons above.
