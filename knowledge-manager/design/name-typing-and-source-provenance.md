# Name typing + source-field provenance

Two model questions. Refs: `model.md` §Building blocks/§Relationships/§"Entity vs literal"/§Promoting
a value/§"Sets = the type"; `km-core/src/model.rs` (`Entity`, `Literal`), `schema.rs resolve_def`,
`resolve.rs effective_sets`; consolidator `entity-promotion.md`, `excel-load-naming.md`.

## Q1 — make `name` a typed data field (any Literal type)?

### Recommendation: no. Keep `name: String`; carry typed values as literal relationships.

### Why name stays a string
- **Identity is `Id`, not name** (`model.rs Entity`; `excel-load-naming.md` §Row name). `name` is the
  human-readable display/lookup handle — required, exactly one per entity.
- **Name is load-bearing for schema resolution.** Def identity is `Set.name`. `resolve_def` does
  `qualified.split_once('.')` → `set_by_name(set_name)` (string index) → match `defines` by name string
  (`schema.rs:221`). A numeric/bool/date name has no qualified form and breaks the `.` separator
  convention (`excel-load-naming.md` already rewrites `.`→`_` in set names to protect this).
- **`by_name`/`set_by_name`/display** all assume a string key.
- Making `name` polymorphic forces every name consumer to handle four types and re-introduces the
  null/“which type” branching the self-typed `Literal` was meant to localize.

### The typed-value need is already met
- `Literal` already spans `Number | String | Bool | Date` (`literal.rs`). Typed values live in
  **relationships**, not in `name`.
- A promoted value-entity **keeps its literal beside it** (`model.md` §Promoting a value: “a promoted
  value-entity still holds its literal so it stays comparable”). So `Unit "8"` can carry a numeric
  literal `8` for range/sort while `name` stays `"8"`. Order/arithmetic is a **computed predicate
  layer** over literals (`predicate.rs`), never over `name`.
- If sortable canonical values are wanted on value-entities, add one typed literal def (e.g. `value`)
  on the value-entity sets — additive, leaves `name` untouched. Today the loader stringifies every cell
  by design (`km-edge-load-excel` lib §faithful: one def = one target type); typed re-parse is the
  app's job on designated columns, and a `value` literal is where a parsed number would land.

## Q2 — round-trip the source fields; separate source vs added defs?

> **Superseded 2026-06-28** (`inter-relation-norms-and-restructure.md`,
> `my-km-data-consolidator/docs/data-model-restructure.md`): option (a) `build-on` was
> implemented then removed. Provenance is now one database node (the row type *is* the
> `source` target); the column manifest is the load report, not a separate set. Q1 (name
> stays a string) stands.

### Nothing is lost; only set-level *distinguishability* is
Derivation/promotion is **purely additive** — raw cells stay string literals, never overwritten
(`entity-promotion.md` §"Keep both"; `enrich.rs` extends `e.rels`). What blurs is the **set schema**:
one set's `defines` accumulates source-column defs + derived literal defs (`unit`, `record-kind`, …) +
promoted link defs (`in-unit`, …) + `asserts-sap-equipment`.

### The faithful source manifest already exists (use it first)
`LoadReport.columns: Vec<(String, Id)>` is the canonical (trimmed, lowercased) source headers,
disambiguated, **in sheet column order** (`km-edge-load-excel` lib §Column-name canonicalization;
`def_of`/`canonical_column`). A faithful export = iterate `columns`, emit each row's
literal for those def ids, in order. **No graph provenance needed** if export is driven by (or persists)
the load manifest. This is the minimal correct answer.

### If the store must be self-describing (export from km alone)
Provenance must live in the graph. Two options:

**(a) Two composed sets via `build-on` — recommended, idiomatic, cheap.**
- `SAP` = source set, `defines` = columns only (loader output, untouched).
- `SAP+` (enriched) with `build-on -> SAP` (a `meta.set` edge on the *set* entity); derived + promoted
  defs authorized on `SAP+`.
- Reassign each row `meta.set -> SAP+`. `effective_sets(row)` = `{SAP+}` ∪ build-on `{SAP}` ∪ universal
  (`resolve.rs`), so the row carries both column + added defs and `validate_all` stays green.
- **Cost ≈ 0 per row**: each row still has its single `meta.set` edge (re-pointed), plus one `build-on`
  edge and one set entity total — the discriminator is not duplicated per row.
- Export source = `defines(SAP)`; added = `defines(SAP+)`. Mirrors the existing three-home layering
  (faithful load / domain derivation / generic promotion). Can extend to three sets
  (source / derived-literal / promoted-link) if that split is ever needed.
- Bonus: the source set becomes a reusable **type** — validate an incoming SAP feed against the source
  schema alone, independently of enrichment.

**(b) `origin` marker on each def — lighter, no set restructure.**
- Tag each def with a meta literal `origin ∈ {source, derived, promoted}` (O(#defs ≈ 150), not O(rows);
  provenance is genuinely a property of the def). Export filters `defines` by `origin`.
- Pick this if all you want is an export filter and you don't want to re-point row `meta.set`.

### Choice
- Immediate round-trip goal → drive export from `LoadReport.columns` (already there).
- Want graph-native, composable source schema → **(a)**.
- Want the lightest graph annotation, no restructure → **(b)**.
- (a) and (b) are O(#defs); per-row cost is ~0 for both, so the old "two sets = a rel per row" worry
  does not apply (build-on lives on the set, not the rows).
