# Step 11 — The worked example (it all comes together)

Files: `src/example.rs` + `design/example.md`. A flow control loop (ISA 5.1): three DCS
points in loop 100, unit 02.

## Understand

**Map code → diagram.**
- Category sets (no defs, pure vocabularies): `Variable`, `Function`, `EngUnit`,
  `SignalType`, `ControlAction`, `Algorithm`, `Conversion`.
- Value-entities (`mk_value`, each assigned to its category set via `meta.set`): `Flow`,
  `Indicate`/`Transmit`/`Control`/`Compute-Convert`, `m3/h`, `4-20mA`/`3-15psi`,
  `direct`/`reverse`, `PID`, `I/P` …
- Schema sets: `Container`(defines `contains`) ← `Unit`, `Loop` build on it;
  `Instrument`(measured-variable/functions/located-in/in-loop) ← `Transmitter`,
  `Controller`, `SignalConverter` build on it; `ProcessSignals`(Data flow).
- Data nodes: `02FIT100` (Transmitter), `02FIC100` (Controller), `02FY100`
  (SignalConverter), `02FV100` (stub); groups `Unit 02`, `Loop 100`.
- These match `design/example.md` tables and the "three points as stored" block.

**Why are `fit`/`fic`/`fy` ids reserved up front (`alloc_id`) before `put`?**
- They're mutually referential: each carries a `Data flow` edge to the next
  (`FIT →PV FIC →OP FY →CO FV`), and `Unit 02`/`Loop 100` `contains` all three. You must
  know an id to build a relationship pointing at it, so the ids are reserved first, then
  the entities are `put` with edges already referencing siblings.

**Follow one `Data flow` + qualifier end to end (PV).**
- `02FIT100` has `rq(d_data_flow, Target::Entity(fic), "PV")` — a `Relationship` with
  `qualifier = Some(Literal::String("PV"))`.
- On `put`, the store's reverse index records, on `02FIC100`, an `Edge { owner: fit,
  def: d_data_flow, qualifier: Some("PV") }`.
- `neighborhood(fic)` / `s.incoming(fic)` surface it; the test
  `reverse_index_yields_incoming_data_flow` asserts owner == `02FIT100` with qualifier
  `PV`.

## Rust

- `&[Id]` slice parameters (`mk_set(..., builds_on: &[Id], ...)`) — borrowed view; pass
  `&[a, b]` or `&vec`.
- `_`-prefixed bindings (`_alg_pi`) silence unused-variable warnings while keeping a
  readable name; `let _ = ca_direct;` explicitly discards/marks-used.
- `#[allow(clippy::too_many_arguments)]` deliberately opts out of one lint for `mk_def`,
  which mirrors a def's fields.

## Try it — add an instrument

Add a new point (e.g. a second transmitter) in `build`, reserving its id with
`alloc_id`. Re-run `validate`. Expect violations until you supply the *required*
`Instrument.located-in` and `Instrument.in-loop` (pointing at `unit02`/`loop100`
instances, not the sets), and keep entity targets inside their `allowed` sets. Fix, then
`validate` returns clean.
