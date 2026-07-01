# Derived-primary set-algebra performance

Follow-on to `selection-and-derived-sets.md` (which chose derived-primary: category
membership is intensional, computed on read). That choice traded O(1) indexed
`members_of_set` reads for on-read predicate evaluation. This note records where the
cost actually lands, the options to recover speed without giving up single-source-of-truth,
and the recommendation.

## Scope note — read path vs. write path

A/C/E/F below optimize the **read path** (derived-set *resolution* and *algebra*). They do
**not** touch the bulk-write phases (load, enrich, promote). A consolidator run dominated by
the **promote** phase (`promoted in …: N value-entities, M links`) is bottlenecked on the
write path, addressed separately under "Write-path indexing". Match the fix to the phase the
profile actually points at.

## Write-path indexing (promote / bulk relate)

Symptom: promote (and any bulk link creation) is slow — far more than the link count
suggests — and is *not* improved by the resolution cache (C), because promote performs no
set resolution.

Root cause:
- The whole mutation API re-puts **entire entities**: `Graph::relate`/`set_one`/`assign_set`
  do `get(owner) → rels.push → put(owner)` (`query.rs`); `promote_all` does
  `e.rels.extend(new) → put(e)` (`km-edge-promote`). There is no incremental edge-add.
- `MemStore::put` (and `RedbStore`) unindex the old entity then re-index the new one.
  Unindexing dropped the entity's id from each bucket it appeared in — and those buckets
  (`reverse`, `set_members`, `by_value`) were `Vec`s, so the drop was `Vec::retain` =
  **O(bucket size)**.
- High-degree buckets are common: a source-table set with 10^5 members, a popular
  value-entity, a `bool` literal (`is-control-valve`, `should-have-sap`) shared by every row.
  Re-homing one such edge is O(10^5) per re-put; a bulk pass over N owners is **O(N·degree)**
  (here ~10^5 owners × ~10^5 → ~10^10 retain steps = the multi-minute promote).

Note: this is pre-existing and **not** caused by the derived-primary switch — that switch
*removed* materialized `meta.set` category writes (fewer edges, not more). The two `bool`
canonical literals it added share large `(def,value)` buckets, a minor contributor at most;
the dominant cost is the source-set membership re-home, which predates it.

Fix (DONE): key the membership/reverse/value buckets by owner id instead of storing a `Vec`
— `set_members`/`by_value` as `BTreeSet<Id>`, `reverse` as `BTreeMap<Id, BTreeMap<Id,
Vec<Edge>>>` (target → owner → edges). Removal becomes **O(log n)**, so each re-put is
O(entity degree) rather than O(Σ target-bucket sizes); the bulk pass drops from O(N·degree)
to ~O(N·log n). `BTreeSet`/`BTreeMap` (vs `HashSet`) also keep reads deterministic without a
sort. Internal to `store/mod.rs`; the `Store` trait API (returns `Vec`) is unchanged, so no
caller or backend signature moves. `redb` shares the same in-memory `Indexes`, so it gets the
fix too.

Deferred follow-on (only if a profile still shows the re-put dominating): add an **incremental
`relate`/`unrelate` to the `Store` trait** that indexes just the changed edge, making a single
link addition O(log n) instead of O(entity degree). Bigger blast radius (trait + both
backends + the `Graph` wrapper); the bucket-keying fix above removes the super-linear term, so
this is a constant-factor refinement, not urgent.

## Status

- **A (schema cache) — DONE.** `resolve_members` resolves `CondSchema` once per top-level
  call and threads it through the recursion (`derive.rs`).
- **C (epoch-keyed resolution memo) — DONE.** `Store::epoch` (bumped on every
  `put`/`delete`) + opt-in `Store::resolve_cache` → `ResolveCache` (`store/mod.rs`);
  `resolve_members` consults it. `MemStore` opts in; other backends default to no cache.
  Consolidator gets it transparently (path-dep on km-core; no consolidator change).
- **E, F — DEFERRED.** Trigger conditions below; suggest implementing when one fires.
- **D, G — rejected by default** (see long-term lens).

## Workload

- Scale: ~10^5 IEs (SAP 55k rows + P&ID 194k rows → reified equipment hubs).
- Shape: **build once, read many.** The store is populated in `equipment::consolidate`,
  then queried read-only across `analyze` / `results` / `enrich`. No mutation interleaves
  the read phase.
- 5 intensional category sets (`confirmed`, `sap_only`, `at_site`, `should_have_sap`,
  `control_valve`), each a single-condition predicate over a literal (`equipment.rs:866`).

## Cost centers (current code)

1. **Schema re-lookup per resolve** — `resolve_inner` calls `condition_schema(store)`
   on every invocation (`derive.rs:269`). That runs 11 `by_name` lookups, each followed
   by an `is_def`/`is_set` typecheck (which itself clones a `rels_by_def` Vec). The
   condition schema is immutable after bootstrap, so this is pure repeated work. A
   `Selection` with two `in_set` clauses pays it twice; any per-set loop multiplies it.
2. **No memoization** — `resolve_members` is pure given an immutable store, but nothing
   caches it. `members_in_all` / `members_with_without` / `Selection::run` re-resolve
   every operand set from scratch each call (`select.rs:46,64,180`). The same 5 category
   sets are resolved repeatedly across phases.
3. **Clone-heavy index reads** — every `Store` accessor returns an owned `Vec`
   (`store/mod.rs:312-330`): `incoming`, `members_of_set`, `rels_by_def`,
   `owners_with_value` all `.clone()` the indexed vector. `conditions_for` clones the
   full reverse-edge Vec of the set, then clones again per `rels_by_def` while decoding.
4. **No reverse `entity → intensional sets`** — `resolve::effective_sets` answers "which
   sets is entity X in" for extensional + build-on only; it cannot see intensional
   membership. Answering it for intensional sets today means evaluating every set's
   predicate. This is the reverse query the perf request names directly.
5. **Negative / empty-positive selections scan the universe** — `Selection::run` with no
   positive clause seeds candidates from `store.ids()` = O(N) (`select.rs:199`); same for
   `evaluate` (`derive.rs:307`). Not hit by the current 5 sets, but a latent O(N) cliff.

The single-condition predicates themselves are cheap: `owners_with_value` is O(result)
via the by_value index. So today's slowness is dominated by (1) constant-factor schema
churn and (2) redundant re-resolution, not by predicate evaluation.

## Options

### A. Cache the condition schema
Resolve `CondSchema` once and reuse (store it on the store, or pass it through the
resolve/select call chain). Removes cost center 1 entirely.
- Win: kills 11 lookups × N-resolves of pure waste. Largest effort/payoff ratio.
- Cost: trivial. Risk: none (schema is immutable post-bootstrap).

### B. Query-scoped memoization
A `HashMap<Id, Rc<Vec<Id>>>` resolution cache threaded through one `Selection`/algebra
call, so a set referenced by several clauses resolves once.
- Win: removes intra-query re-resolution.
- Cost: low. Risk: none (scoped to one read).

### C. Store-level resolution cache with epoch invalidation
Keep an `epoch: u64` on the store, bumped on every `put`/`delete`. Memoize
`resolve_members(set) -> (epoch, Vec<Id>)`; a hit on the current epoch is O(1). Because
the read phase performs zero mutations, every repeat resolve after the first is free —
this restores materialized-primary read speed **without** materializing anything or
adding a second source of truth (the cache is derived and self-invalidating).
- Win: the main fix for "build once, read many." Repeat membership reads → O(1).
- Cost: moderate (cache + epoch plumbing). Risk: low; correctness reduces to "bump epoch
  on every mutation," which `put`/`delete` already centralize.

### D. Incrementally materialized membership (maintained cache)
Maintain a reverse `set → members` index for intensional sets, updated on `put`/`delete`.
Needs a condition index (option E) to know which sets a changed `(def, value)` affects, so
only touched sets re-evaluate. This is the materialized-primary end-state the design
permits "if one is unambiguously derived from the other" — here the literal stays
canonical and membership is a maintained cache.
- Win: O(1) reads even under heavy mutation interleaving; fully indexed set algebra.
- Cost: high (incremental view maintenance is the fiddly part flagged as "C/medium" in
  the parent doc). Risk: maintenance bugs reintroduce drift the redesign removed.
- Verdict: over-engineered for the current build-once/read-many shape; C gets the same
  read speed at a fraction of the risk. Revisit only if mutation starts interleaving reads.

### E. Condition reverse index `(def, value)` / `(def, target) → sets`
Index the stored conditions so that, given an entity's relationships, the candidate
intensional sets are found in O(rels) instead of by scanning every set's predicate.
- Win: answers the explicit reverse query "which sets does this entity belong to" for
  intensional sets; also the enabling index for D's incremental maintenance.
- Cost: low–moderate (one more `HashMap` maintained alongside `by_value`). Risk: low.

### F. Bitset / roaring-bitmap algebra
Represent membership as roaring bitmaps (or sorted id slices) so intersection / union /
difference are bitwise merges instead of `HashSet::retain(contains)` loops with per-op
allocation (`select.rs:191-214`, `derive.rs:299-313`).
- Win: this is the direct answer to "set algebra highly optimized" at 10^5 scale —
  large-set intersections become near-linear scans over compressed words, with far less
  allocation and memory than `HashSet<Id>`.
- Cost: moderate (id↔bitmap mapping; a dep like `roaring`). Risk: low, localized to the
  algebra layer; pairs naturally with C/D (cache bitmaps instead of Vecs).

### G. Borrow-not-clone store reads
Change accessors to return `&[..]` / iterators instead of cloned `Vec`s, removing cost
center 3.
- Win: removes per-call allocation across all index reads.
- Cost: moderate (lifetime plumbing through the `Store` trait; affects every backend and
  caller). Risk: medium (API churn). Lower priority than A/C — caching makes most of these
  reads not happen at all.

## Recommendation

Order **A → C → F**, with **E** when the reverse query is needed and **D/G** deferred.

- **A now** — trivial, removes the biggest pure-waste constant factor; do it regardless.
- **C** is the primary fix. The consolidator's build-once/read-many shape means an
  epoch-keyed resolution memo makes every repeat membership read O(1), recovering
  materialized-primary read speed while keeping derived-primary's single source of truth.
- **F** if profiling after A+C shows intersection/union dominating — it is the specific
  lever the request asks for ("set algebra highly optimized") and composes with C (cache
  bitmaps).
- **E** the moment "which sets is entity X in?" becomes a real query over intensional sets
  (explorer, enrichment), and as the prerequisite for D.
- **D** only if mutation begins interleaving reads, making cache misses frequent; otherwise
  C delivers the same read performance at much lower risk.
- **G** opportunistically; caching (C) removes most of the clones it targets.

All A–F are km-core (generic substrate) changes; the consolidator keeps defining its 5
sets intensionally and gains the speed for free. Keep the per-set extensional/intensional
choice — none of these flips it.

## Long-term lens — does each earn a permanent place in a generic substrate?

Judged not by "fixes the consolidator" but by: genericity across problem shapes,
forever-maintenance cost, and fit with the locked principles ("views derived not stored",
"indexes are a regenerable cache", `Store` trait implementable by multiple backends).

- **A — unconditional yes.** No API surface, no semantic change, just deletes repeated
  work. Nothing to regret later.
- **C — yes; it is the principle, not a hack.** "Memoize a pure derivation, invalidate on
  an epoch" is exactly the doc's "indexes are a regenerable cache" stance applied to
  resolution. Generic to every intensional-set workload. Honest limit: a *global* epoch
  invalidates all memo entries on any write, so it is read-phase-optimal and degrades to a
  no-op under write-interleaved reads — but it is never *wrong* and never a net negative.
  That degradation is the boundary with D, not a defect.
- **E — yes, but demand-driven.** "Which categories does this entity fall into?" (reverse
  classification) is a fundamental KG query; a general substrate will want it (explorer,
  enrichment, LLM context-building). Bounded, well-understood index. Build it when a second
  consumer actually needs the reverse direction — not speculatively. Also the prerequisite
  for D.
- **F — conditional; reversible by construction.** Roaring bitmaps are the canonical
  long-term representation for id-set algebra (Lucene/Druid lineage), and set algebra *is*
  km-core's core operation — so this is plausibly the "right" substrate, not a mere
  optimization. But the payoff scales with N and algebra complexity; at 10^3 it is noise.
  A general substrate must not tax small-N problems. Keep it *behind* the `Vec<Id>` API so
  it stays an internal representation that profiling can justify or revert. Earn it; don't
  assume it.
- **D — skeptical; the long-term trap.** Generic incremental view maintenance over the full
  predicate language (negation + recursive `InSet` chains, where one literal edit cascades
  through dependent sets) is a permanent correctness liability — exactly the drift the
  redesign removed, re-entering through the maintenance path. Only justified in a
  simultaneously write-heavy *and* read-heavy regime, which is narrow. If it ever lands, make
  it **per-set opt-in**, never a blanket policy. Default: don't.
- **G — net-negative for genericity; do not.** Owned-`Vec` returns are a deliberate
  affordance, not laziness: they let the `Store` trait be implemented by backends that
  *don't* hold everything in memory (disk/redb-paged, remote, lazily-materialized). Borrowed
  `&[..]` returns force every backend to own the data in the shape the index wants,
  shrinking the set of possible backends. Caching (C) removes most of the clones G targets
  anyway. If allocation still bites, reach for `Arc<[Id]>`/`Cow`, not lifetime-threaded
  borrows.

Net: **A and C are generically worth it now** (C *is* the principle). **E and F are worth
it on demand**, gated by a second consumer / a profile — and F stays reversible behind the
existing API. **D and G are long-term liabilities** to avoid by default.

## Why "defer E/F" survives the "but the goal *is* genericity" objection

The objection is fair: "wait for a second consumer" is circular when building for future
consumers is the point. The real case rests on an asymmetry and two under-determinations,
not on YAGNI.

**The asymmetry (cheap to wait, expensive to guess).** A + C + the existing `Vec<Id>` API
are a *stable seam*. E and F can be added later as drop-in accelerators **without changing
any call site** — the query API and return types don't move. So deferral costs ~nothing to
undo. Building now, against imagined query shapes and data scales, bakes assumptions into
the substrate; a generic API built against *one* example is often *less* generic than a
minimal core plus deferred accelerators. For a substrate meant to generalize, that failure
mode is worse, not better. Precedent: Lucene/Druid did not ship roaring bitmaps on day one —
they were added later, behind stable APIs, once profiles demanded them.

**E is under-determined by the predicate language.** A `(def,value)→sets` index reverses
`HasValue` cleanly, but the language also has `LinkedTo` (needs `(def,target)→sets`),
recursive `InSet`, and **negation** — and "in a set by *not* having X" cannot be enumerated
by a forward value lookup at all. So the *complete* reverse query is not one HashMap; it is
a candidate-generate-then-verify scheme whose right shape only becomes clear once real
predicates exist. Build the easy partial index speculatively and you get one that silently
handles `HasValue` and misses negation/`InSet` — a correctness footgun worse than not having
it. The cheap version is the wrong version; the right version needs evidence.

**F is under-determined by the id model.** Roaring's win needs *dense u32 ordinals*. km-core
ids are sparse `u64` from `IdGen`, so real bitmap efficiency requires either making ids dense
ordinals (constrains allocation/deletion → tombstones/compaction) or maintaining an
id↔ordinal bimap — another index in lockstep, and a change to the `Snapshot` on-disk format.
That part is **not** cheaply reversible once data is serialized. "Reversible behind the API"
holds for the *algebra layer* (HashSet→bitmap inside `evaluate`), but the structural win that
also accelerates E and repeated algebra touches `id.rs` + persistence — and you cannot tune
an id model you have not yet profiled at a real N.

**When the answer flips:** if a concrete second problem is already in view with known query
shapes (does it need the reverse direction? with negation?) and a known scale (is N large
enough that bitmap algebra beats HashSet?), then E — and possibly F's id-model decision —
become worth doing *with* that evidence rather than against a guess. Absent that, A + C now,
E/F behind the seam, is the genuinely more generic path, not the more timid one.

## Deferred accelerators — suggest implementing when (E, F)

Forward-looking triggers so future development (human or agent) can recognize when a
deferred item has become worth it and *propose* it. Each is a checkable signal, not a date.
If a trigger fires, open the work referencing this section; if none has, the A+C seam is
sufficient and adding E/F is premature.

### E — condition reverse index `(def,value)`/`(def,target) → sets`
Capability: given an entity, list the intensional sets it belongs to in O(rels) instead of
evaluating every set's predicate. Also the prerequisite index for D.

Suggest implementing when **any** holds:
- A consumer needs the **reverse** "which categories/sets is entity X in?" over *intensional*
  sets (not just extensional/build-on, which `resolve::effective_sets` already covers) — e.g.
  the explorer's entity-detail view, enrichment tagging, or LLM context-building.
- The number of intensional sets grows past a handful, so answering the reverse question by
  scanning every set's predicate is itself the cost.
- D is being seriously considered (E is its enabling index).

Design note before building: the predicate language has `LinkedTo` (needs `(def,target)`),
recursive `InSet`, and **negation**. Negative membership ("in set by *not* having X") does not
forward-index — so E must be a *candidate-generate-then-verify* scheme, not a single exact
HashMap. Do not ship a `HasValue`-only partial index that silently drops negated/`InSet`
conditions; that is a correctness footgun. Wait for real predicates to fix the shape.

### F — bitset / roaring-bitmap set algebra
Capability: replace `HashSet<Id>` intersection/union/difference (`select.rs` `members_in_*`
/ `Selection::run`; `derive.rs` `evaluate`) with bitwise merges; optionally cache membership
as bitmaps (composes with C — cache bitmaps instead of `Vec<Id>`).

Suggest implementing when **all** of 1–2 plus either 3a or 3b hold:
1. A profile (after A+C) shows set-algebra intersection/union/difference — not predicate
   evaluation, not I/O — as the dominant cost.
2. Working-set N is large (≳10^4–10^5 ids per set) where compressed-bitmap algebra beats
   hashed membership tests; at ≲10^3 it is noise and not worth the dependency.
3a. The algebra layer alone is the hot path → adopt bitmaps **internally inside `evaluate`/
   `Selection`**, behind the unchanged `Vec<Id>` API. Cleanly reversible. Prefer this first.
3b. The win must also accelerate E/repeated algebra via a *stored/maintained* bitmap → this
   touches the id↔ordinal mapping and the `Snapshot` on-disk format, which is **not** cheaply
   reversible. Gate this harder; decide the id model deliberately.

Id-model note: km-core ids are already monotonic and dense from `USER_ID_START` (`id.rs`),
which is *favorable* for roaring (near-contiguous ordinals) — so F is more feasible than a
sparse-id store. The remaining cost is deletions punching holes (tombstone/compaction) and,
for 3b, evolving the serialized format. The internal-only path (3a) sidesteps both.
</content>
