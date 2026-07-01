# Worked Example — A Flow Control Loop (ISA 5.1)

Three DCS points from the **same loop 100 in unit 02**, as **stored entities** under the entities-and-relationships model (no properties; a relationship's target is an entity or a literal).

## ISA 5.1 decoding
Tag = `[area][first-letter][succeeding letters][loop number]`.
- **first letter** = measured/initiating variable. `F` = Flow.
- **succeeding letters** = functions. `I` = Indicate, `T` = Transmit, `C` = Control, `Y` = Relay/Compute/Convert.
- shared `02` = area/unit; shared `100` = loop number → all three are one control loop.

| Tag | Parse | Meaning | Role in loop |
|-----|-------|---------|--------------|
| `02FIT100` | 02 / F / IT / 100 | Flow Indicating **Transmitter** | measures flow → emits PV |
| `02FIC100` | 02 / F / IC / 100 | Flow Indicating **Controller** | PV vs setpoint → emits output |
| `02FY100`  | 02 / F / Y  / 100 | Flow **Compute/Convert** relay | converts controller output (I/P) → drives final element |

Signal chain: `02FIT100 →(PV) 02FIC100 →(OP) 02FY100 →(CO) 02FV100` (valve, out of scope here).

## Stored record format
Every entity is the same shape. `name` is a field; everything else is a relationship.
```
<id>  name="..."
  rels:  Set.type [ "qualifier" ] -> target        (target = entityId or a literal)
```
A node stores **which sets apply** as `meta.set` relationships. A set "builds on" another the same way. Effective sets = a node's `meta.set` relationships, transitively closed.

## Bootstrap: the meta set (describes sets & definitions)
```
set:meta   name="meta"
  rels: meta.kind -> "set"
        meta.defines -> def:meta/kind          (target literal: string)
        meta.defines -> def:meta/target-type   (literal | entity-set)
        meta.defines -> def:meta/target-level  (instance | schema)
        meta.defines -> def:meta/cardinality   (one | many)
        meta.defines -> def:meta/required      (bool)
        meta.defines -> def:meta/constraint    (range / allowed values / allowed set)
        meta.defines -> def:meta/set           (rel-def: a set that applies)
        meta.defines -> def:meta/defines       (rel-def: holds a definition)
        meta.defines -> def:meta/target        (rel-def: a rel-def's target set)

literal types: number, string, bool, date     (primitive, built in)
```
The meta set is defined in terms of itself; it is the fixed point the rest builds on.

## Value / concept entities (referenced, shared)
```
var:Flow    fn:Indicate  fn:Transmit  fn:Control  fn:Compute-Convert
sig:4-20mA  sig:HART     sig:3-15psi  eu:m3/h
grp:Unit-02   name="Unit 02"   (meta.set -> set:Group)
grp:Loop-100  name="Loop 100"  (meta.set -> set:Group)
```

## Sets as stored entities

### Instrument (full detail — each definition is itself an entity)
```
set:Instrument  name="Instrument"
  rels: meta.kind -> "set"
        meta.defines -> def:Instrument/measured-variable
        meta.defines -> def:Instrument/functions
        meta.defines -> def:Instrument/located-in
        meta.defines -> def:Instrument/in-loop

def:Instrument/measured-variable  name="measured-variable"
  rels: meta.kind=def  meta.target-type -> entity-set Variable  meta.cardinality -> one
def:Instrument/functions          name="functions"
  rels: meta.kind=def  meta.target-type -> entity-set Function  meta.cardinality -> many
def:Instrument/located-in         name="located-in"
  rels: meta.kind=def  meta.target-type -> entity-set, meta.target -> set:Unit
        meta.target-level -> instance  meta.cardinality -> one  meta.required -> true
def:Instrument/in-loop            name="in-loop"
  rels: meta.kind=def  meta.target -> set:Loop  meta.target-level -> instance
        meta.cardinality -> one  meta.required -> true
```

### Transmitter / Controller / SignalConverter (build on Instrument via meta.set)
```
set:Transmitter  name="Transmitter"
  rels: meta.set -> set:Instrument                      (build-on)
        meta.defines -> def:Transmitter/range-low       (target literal: number)
        meta.defines -> def:Transmitter/range-high      (literal: number)
        meta.defines -> def:Transmitter/eng-unit        (entity-set EngUnit)
        meta.defines -> def:Transmitter/output-signal   (entity-set SignalType)

set:Controller   name="Controller"
  rels: meta.set -> set:Instrument
        meta.defines -> def:Controller/setpoint         (literal: number)
        meta.defines -> def:Controller/output           (literal: number)
        meta.defines -> def:Controller/control-action   (entity-set, ∈ direct,reverse)
        meta.defines -> def:Controller/algorithm        (entity-set, ∈ PID,PI,P)

set:SignalConverter  name="SignalConverter"
  rels: meta.set -> set:Instrument
        meta.defines -> def:SignalConverter/conversion     (entity-set, ∈ I/P,P/I,compute)
        meta.defines -> def:SignalConverter/input-signal   (entity-set SignalType)
        meta.defines -> def:SignalConverter/output-signal  (entity-set SignalType)
```

### ProcessSignals (universal set — applies to all instruments)
```
set:ProcessSignals  name="ProcessSignals"
  rels: meta.defines -> def:ProcessSignals/Data flow
def:ProcessSignals/Data flow  name="Data flow"
  rels: meta.kind=def  meta.target -> set:Instrument  meta.target-level -> instance
        meta.cardinality -> many                       (qualifier = signal role)
```

### Group (grouping — defines `contains`)
`contains` is an **ordinary domain def**, not core/bootstrap vocabulary — owned here by a `Group` set
that the unit/loop entities are assigned to (`design/decisions.md` §Containment). No bare, undefined
`contains`: like every relationship it is `Set.name` and references a definition.
```
set:Group  name="Group"
  rels: meta.defines -> def:Group/contains
def:Group/contains  name="contains"
  rels: meta.kind=def  meta.target -> set:Instrument  meta.target-level -> instance
        meta.cardinality -> many
```

## The three points as stored
```
n:02FIT100  name="02FIT100"
  rels: meta.set -> set:Transmitter
        meta.set -> set:ProcessSignals
        Instrument.measured-variable      -> var:Flow
        Instrument.functions              -> fn:Indicate
        Instrument.functions              -> fn:Transmit
        Transmitter.range-low             -> 0          (literal)
        Transmitter.range-high            -> 100        (literal)
        Transmitter.eng-unit              -> eu:m3/h
        Transmitter.output-signal         -> sig:4-20mA
        Instrument.located-in             -> grp:Unit-02
        Instrument.in-loop                -> grp:Loop-100
        ProcessSignals.Data flow ["PV"]   -> n:02FIC100

n:02FIC100  name="02FIC100"
  rels: meta.set -> set:Controller
        meta.set -> set:ProcessSignals
        Instrument.measured-variable      -> var:Flow
        Instrument.functions              -> fn:Indicate
        Instrument.functions              -> fn:Control
        Controller.setpoint               -> 60         (literal)
        Controller.control-action         -> reverse
        Controller.algorithm              -> PID
        Controller.output                 -> 0          (literal)
        Instrument.located-in             -> grp:Unit-02
        Instrument.in-loop                -> grp:Loop-100
        ProcessSignals.Data flow ["OP"]   -> n:02FY100
        (incoming: n:02FIT100 Data flow "PV")

n:02FY100   name="02FY100"
  rels: meta.set -> set:SignalConverter
        meta.set -> set:ProcessSignals
        Instrument.measured-variable      -> var:Flow
        Instrument.functions              -> fn:Compute-Convert
        SignalConverter.conversion        -> I/P
        SignalConverter.input-signal      -> sig:4-20mA
        SignalConverter.output-signal     -> sig:3-15psi
        Instrument.located-in             -> grp:Unit-02
        Instrument.in-loop                -> grp:Loop-100
        ProcessSignals.Data flow ["CO"]   -> n:02FV100   (final element, stub)
        (incoming: n:02FIC100 Data flow "OP")

grp:Unit-02   rels: meta.set -> set:Group ; Group.contains -> n:02FIT100 ; Group.contains -> n:02FIC100 ; Group.contains -> n:02FY100
grp:Loop-100  rels: meta.set -> set:Group ; Group.contains -> n:02FIT100 ; Group.contains -> n:02FIC100 ; Group.contains -> n:02FY100
```

## Effective sets (resolved from stored meta.set relationships)
- `02FIT100` direct {Transmitter, ProcessSignals} → +Instrument (via Transmitter) = **{Transmitter, Instrument, ProcessSignals}**.
- `02FIC100` → {Controller, Instrument, ProcessSignals}.
- `02FY100`  → {SignalConverter, Instrument, ProcessSignals}.

## What this exercises
- **No properties** — `measured-variable -> Flow` (entity) and `range-high -> 100` (literal) are the same kind of thing: relationships. Entity targets are shared concepts; literals are inert scalars.
- **Value-entities carry knowledge** — `eu:m3/h`, `sig:4-20mA`, `var:Flow` are reusable nodes you can describe and query by, not repeated strings.
- **Sets are the type, stored as data** — definitions are entities held by `meta.defines`; applied sets are `meta.set` relationships, resolved by one transitive walk.
- **Namespacing** keeps `Transmitter.output-signal` and `SignalConverter.output-signal` distinct.
- **Directed Data flow** carries the signal role as the qualifier (PV/OP/CO); the reverse index yields each node's incoming signals.
