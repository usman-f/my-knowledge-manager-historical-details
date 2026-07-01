# Inter-relation norms — why the model drifted, and the guardrails

Retrospective + generic principles. Triggered by the consolidator restructure
(`my-km-data-consolidator/docs/data-model-restructure.md`). Refs: `model.md`,
`decisions.md` (§Namespacing, §Keep literals, §Qualifier, §Provenance edge layer).

## The thesis we drifted from
A graph/inter-related store answers **provenance, sharing, and type** with **edges to shared
nodes** — not by partitioning the schema, copying values per row, or minting parallel nodes for
one concept. Every restructure symptom below is the same root: reaching for *new structure* where
an *edge to an existing node* was the answer.

## Symptoms observed (consolidator)
1. **Two nodes for one concept.** The loader mints a schema **set** `SAP` (rows are `meta.set`
   members; holds column defs) *and* a separate source **entity** `SAP` in a `sources` set (rows
   link via `source`). One database, two `SAP` nodes — the relational table-vs-row habit leaking
   into a graph where the database is one node.
2. **Provenance solved by schema partition.** "Which columns were present at import" was answered
   by per-record set membership + a `build-on RecordEnrichment` apparatus + namespaced
   `SAP.<column>` defs — structure whose only job was to keep the source column manifest isolable.
   That fact is a property of the **database node** (its `defines`), reachable from any record by
   one `source` edge. The partition was unnecessary.
3. **Redundant denormalization.** Promotion kept **both** a literal `unit "8"` *and* a link
   `in-unit -> Unit 8` on every record/IE ("keep both"). The relational instinct to copy a value
   into every row, versus the graph instinct to point at the one shared node. The comparability
   literal belongs **once** on `Unit 8` (`model.md` §Promoting a value already says this), not on
   every citing record.
4. **Modeling against a hypothesized conflict.** The source-vs-enrichment split was justified by
   the worry that two databases might assert different units for the same equipment. Probes later
   showed cross-source agreement ~93% and the residual conflicts are FK-authoritative (already
   flagged, not split). The feared problem was rare and already handled — structure was added
   before the problem was demonstrated.

## Root causes (how good norms got lost)
- **Faithful-load-first, then bolt-on layers.** Load wrote literals-only; every later need
  (sharing, dedup, provenance) was added as an *additive layer over the literals* instead of
  re-modeling the value as an entity. "Keep both" is the fingerprint of layering instead of
  modeling.
- **Provenance treated as a schema concern, not a data edge.** Once you decide provenance lives in
  the schema, you get namespaced defs, parallel sets, and build-on chains. The store already had
  the right primitive: an edge to a source node.
- **Concepts not unified.** Two things that always co-denote (the `SAP` set and the `SAP` source
  entity) were never merged. A set *is* an entity (`model.md`); nothing forced the split.
- **Worry-driven structure.** A plausible future conflict was hardened into present structure
  before any instance of it existed in the data.

## Guardrails (apply to every future edge/domain model)
1. **Provenance is an edge to a source node — never a schema partition.** "Where did this come
   from / which fields did it have" = follow `source` to the database node and read *its* manifest.
   Do not encode origin in def namespaces, per-record set membership, or build-on chains.
2. **One concept = one node.** If two entities always co-denote, merge them. The database is a
   single first-class node; it can simultaneously be the **type** of its records (its `defines` =
   the column manifest) and the thing they cite as `source`.
3. **A value that can be an entity is stored once, as the entity.** Promotion **replaces** the
   record's literal with the entity link; it does not keep both. The sortable/comparable literal
   lives on the value-entity (`Unit 8` holds literal `8`), not duplicated on every citer. Keep a
   literal on the record only for genuinely instance-unique scalars (raw tag, measurements, dates,
   ids, free text) that are never promoted.
4. **Don't model against a conflict you haven't seen.** Model the simple shared-target case first.
   When a real conflict appears, annotate the edge (qualifier: stage/confidence/source-claim) or
   flag it — adding *data*, not a new schema partition.
5. **Smell test.** If a feature adds a parallel set/namespace/duplicate-node to express something a
   single edge could express, that is the drift. Ask "would this be an edge in a graph?" before
   adding structure.

## Consistency with settled decisions
- Reinforces §Namespacing (share a def by putting it in a base set both build on — *not* by
  per-source copies) and §Keep literals (literals are the ground floor; promote sparsely — the
  promoted entity is the value, the record need not also carry the string).
- The qualifier (§Qualifier) is the native home for per-edge stage/confidence
  (`parsed`/`enriched`/`decided`), symmetric with the claim/provenance edge layer (§Provenance).
- No km-core change implied: all guardrails are uses of existing primitives (entities, edges,
  qualifiers, `defines`). The drift was in *application* modeling, not core capability.
