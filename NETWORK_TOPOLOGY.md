# RUNS Network Topology

> **Layer**: Wiring (Networks)
> **File extension**: `.runs`
> **Status**: Design stage — no reference scheduler or validator exists yet. Precise
> enough to implement from; the first implementation is expected to correct it.

## Purpose

This specification defines the Network: the wiring layer that connects Records to
Processors and constitutes one game tick. Processors define computation
([DIGS](./DIGS_EXPRESSION_LANGUAGE.md)); Records define state
([Record Schema](./RECORD_SCHEMA.md)); Networks define how data flows between them —
which Processor reads which Fields, where its outputs land, and how entity
collections are routed through guarded arcs.

Every decision in this specification is driven by one question: *can a solo developer
in a century implement this from scratch, with nothing but this document?*

---

## The Model

### A bipartite graph

A Network is a bipartite directed graph with two disjoint node kinds: Records
(places, nouns) and Processors (transitions, verbs). Every arc connects a Record to a
Processor or a Processor to a Record. Data flows strictly **Record → Processor →
Record** — never Record → Record (no verb, so no transformation) and never
Processor → Processor (no noun, so no data at rest). This is not a convention; it is
the mandatory structural invariant. A Network is the only thing that knows both
sides: Records and Processors are mutually opaque, and neither knows where its data
came from or goes.

The natural reading is a **colored Petri net**: places = Records, tokens = Field
values (the colors), transitions = Processors, guarded arcs = gates.

A Network references the Records and Processors it wires **by ID** and never embeds
them. It is pure wiring.

### The single-assignment law

The law of the whole system, at every scale: **write once, never overwrite; pass
shared things hand-to-hand.**

- A place is written **once** per tick. A "changed" Record is a fresh value in a
  fresh place — a new *version* — never an in-place overwrite.
- A **shared resource** — a PRNG, an accumulator, a spawn queue, an object pool — is
  **threaded**: passed hand-to-hand through the Processors that touch it as a chain
  of distinct versions, `prng_v0 → P_a → prng_v1 → P_b → prng_v2`. Threading is
  ordinary read-after-write wiring, nothing more.
- The wiring syntax lets a step reuse a Record's name (`objects =
  spacewar:collision_detect(objects, consts)`). That is shorthand, not mutation: each
  rebinding is a new version, and textual order resolves which version a later
  reference means — exactly like `let` shadowing in a DIGS body. A validator lowers
  the wiring to explicit versions (SSA) and checks the law mechanically.

Because places are single-assignment, the only dependency that can exist is
read-after-write. One tick's dataflow is therefore a **DAG**, always.

### Execution order is derived, never authored

There is no execution-order primitive in RUNS — no phase list, no priorities, no
scheduling directives. The author declares *wiring*; order is whatever the dataflow
DAG implies:

- A step that reads version N of a place runs after the step that produced
  version N. That is the entire ordering rule.
- Two steps **incomparable** in the DAG touch disjoint data — neither reads what the
  other writes — so they **commute**. The net is **confluent**: every valid
  topological order produces bit-identical results. A minimal runtime executes steps
  top-to-bottom; a performance runtime runs incomparable steps in parallel; both are
  equally conforming because the results cannot differ.
- Where original behavior genuinely depends on order — ship 1 acts before ship 2,
  consuming the PRNG first — that order is *data*: thread the shared resource and the
  ordering becomes a derived fact of the chain. If a Network seems to need an
  *authored* order that the data does not imply, the model says: some shared thing is
  being overwritten instead of threaded. Thread it.

### Where determinism holds: deterministic regions

Confluence as stated above requires that the Processors themselves are deterministic
— outputs a pure function of declared inputs. That is guaranteed for every Processor
whose body is DIGS, by the language's core. It is *not* guaranteed for Processors
whose job is to touch the platform (§The Platform Boundary): rendering, audio,
input, timing, transport are inherently platform-coupled, and the standard
deliberately does not specify a language for their bodies.

The determinism guarantee therefore scopes to a **deterministic region**: a maximal
subgraph all of whose Processors have DIGS bodies, bounded by the Records at its
edge. Within such a region, determinism is *structural* — it follows from
single-assignment plus partitioning guards (§Guards), with no further discipline
needed. Outside it, the same petri-net topology holds but confluence is not claimed.

A well-built game keeps all of its rules — everything that decides what happens —
inside one deterministic region, and pushes platform contact to the edges. The
conventions layer names this pattern and its consequences (the
[RUNS Library](https://github.com/enduring-game-standard/runs-library) patterns
document); the base spec only defines the structure that makes it possible.

### The tick loop: bounded inside, unbounded across

Every Processor is total — each invocation terminates (DIGS core). Every collection
fold in the wiring is bounded by the collection it folds. So one tick always
terminates. The **only** unbounded loop in all of RUNS is the cross-tick feedback:
`state:` places carry this tick's final versions into the next tick, forever. That
loop lives in the wiring, visible in the topology — never inside a body.

The cross-tick `state:` places may grow without bound (an unbounded list appended to
every tick) and the game may run without bound. Both are legal and fully
deterministic. Two consequences follow, game bugs for tooling to catch rather
than language violations: a Network *can* livelock across ticks (a feedback cycle
whose exit guard never fires), and a `state:` list *can* leak — deterministically,
reproducibly, identically on every machine.

### The two composition planes

There are two ways Processors compose, seamed at the Record:

- **Body-plane**: a DIGS body calls a sub-Processor (`let r = ns:proc(x)`). The
  intermediates are anonymous body-locals — record-shaped *values*, never Records.
  From outside, the whole call DAG collapses into one opaque transition. Governed by
  DIGS's totality rules, not by the bipartite rule.
- **Network-plane**: the bipartite graph proper, where every intermediate is a
  Record — the only kind of intermediate that can persist across ticks, cross the
  platform boundary, or be dispatched on by guards.

Bundling and unbundling are exactly the act of crossing the seam. Promoting a
body-local to a Record *unbundles* — the intermediate becomes inspectable,
dispatchable, persistable wiring. Demoting a Record to a body-local *bundles* — it
disappears inside a transition. "No Processor → Processor" is never violated:
network-plane composition always lands in a Record; body-plane composition is
interior to one transition.

"Network" is **scale-free**. The whole game is one Network; every bundled chunk is
also a Network (a **Sub-Network** when viewed from outside as a Processor). Fully
unbundled, the game is a single flat bipartite graph; there is no privileged
"sub-network" entity — only regions at different zoom levels. Tooling must treat
free bundling/unbundling as the normal way to view and edit the graph.

---

## Network Declaration

### Root Network

A root Network declares one complete game tick — the unit the host invokes:

```
network spacewar:game_tick

  requires:
    tick:        spacewar:tick_input
    controls_1:  spacewar:player_controls
    controls_2:  spacewar:player_controls

  produces:
    frame:       spacewar:render_frame

  state:
    objects:      spacewar:object[24]
    prng:         spacewar:prng_state
    result:       spacewar:match_result
    consts:       spacewar:game_constants
    star_catalog: spacewar:star_catalog
      source: spacewar:data/star_catalog

  local:
    spawn_requests: spacewar:spawn_request[max: 24]

  wiring:
    # ... (steps; see §Wiring)
```

| Block | Required | Contents | Lifetime |
|-------|----------|----------|----------|
| `requires:` | Yes | Inbound boundary Records — written by the host before each tick | One tick |
| `produces:` | Yes | Outbound boundary Records — read by the host after each tick | One tick (snapshot) |
| `state:` | No | Persistent Records — each tick starts from the previous tick's final versions | Cross-tick |
| `local:` | No | Tick-local working Records — start each tick at their schema defaults | One tick |
| `wiring:` | Yes | The dataflow steps | — |

A place declared in `state:` may also be listed in `produces:` — the host reads it
after each tick, but the game owns and persists it.

### Sub-Network

A Sub-Network is a Network used as a Processor: typed `inputs:` and `outputs:` at
its boundary, wiring inside.

```
network spacewar:ship_step
  inputs:
    object:         spacewar:object
    controls:       spacewar:player_controls
    consts:         spacewar:game_constants
    prng:           spacewar:prng_state
    spawn_requests: spacewar:spawn_request[max: 24]
  outputs:
    object:         spacewar:object
    prng:           spacewar:prng_state
    spawn_requests: spacewar:spawn_request[max: 24]

  wiring:
    - (object, prng) = spacewar:rotation_update(object, controls, consts, prng)
    - object = spacewar:gravity_apply(object, consts)
    - object = spacewar:thrust(object, controls, consts)
    - object = spacewar:wrap_position(object)
    - (object, spawn_requests) = spacewar:torpedo_launch(object, controls, consts, spawn_requests)
    - (object, prng) = spacewar:hyperspace_check(object, controls, consts, prng)
```

From the call site, a Sub-Network is indistinguishable from a Processor. Note the
threading in the example: `object`, `prng`, and `spawn_requests` are each passed
hand-to-hand down the pipeline — six links of a thread, written with name reuse, and
the order of these steps is exactly the order the threads imply, nothing more.

### Place declarations

References in `requires:`, `produces:`, `state:`, `local:`, `inputs:`, and
`outputs:` blocks share one form:

```
name: qualified_type
name: qualified_type[N]
name: qualified_type[max: N]
name: qualified_type[]
```

Collection notations and their meaning — including that unbounded `[]` is legal and
a declared bound is an opt-in memory contract — are defined in
[Record Schema §Lists](./RECORD_SCHEMA.md#lists-and-the-memory-contract).

### Data sources

A `state:` place may declare a build-time data source:

```
state:
  star_catalog: spacewar:star_catalog
    source: spacewar:data/star_catalog
```

`source:` is declarative metadata for the build tool: it names the artifact whose
contents populate this place before the first tick — an AEMS Manifestation, a data
file (WAD lump, ROM segment), an embedded table. It has no runtime semantics. How
raw bytes map to Fields is defined by a companion format-specification artifact, not
by this document.

---

## Wiring

The `wiring:` block is a list of **steps**. Each step binds the outputs of one
invocation to named places. Steps are dataflow declarations, not an execution
schedule: the order of the list resolves name reuse into versions (§The
single-assignment law), and execution order is derived from the resulting DAG.

### Invocation step

```
- objects = spacewar:collision_detect(objects, consts)
- (objects, result) = spacewar:check_restart(objects, result, consts)
```

The right side invokes a Processor or Sub-Network. Arguments are positional against
the target's declared `inputs:`. The left side binds the target's declared
`outputs:`, positionally: a single name, or a parenthesized tuple matching the
output count. Every output must be bound — a Processor's output cannot be silently
dropped (bind it to a `local:` place if it is genuinely unused at this call site,
and say why in a comment).

Argument and binding types must match the target's declarations exactly.

### Guarded step

```
- guard: tick.is_substep
  objects = physics:integrate(objects, tick)
```

The guard is a DIGS boolean expression over in-scope places. If true, the step runs.
If false, every place the step would bind passes through unchanged — the guarded
step is sugar for a two-arm partition whose `else` is identity. Either way the bound
places get exactly one new version, so single-assignment is preserved.

### Dispatch step

Dispatch is the construct for *for each entity, route by state* — a **guarded,
threaded fold** over a collection:

```
- (objects, prng, spawn_requests) = dispatch objects threading (prng, spawn_requests):
    arcs:
      - guard: .state == spacewar:ship and slot == 0
        (., prng, spawn_requests) = spacewar:ship_step(., controls_1, consts, prng, spawn_requests)

      - guard: .state == spacewar:ship and slot == 1
        (., prng, spawn_requests) = spacewar:ship_step(., controls_2, consts, prng, spawn_requests)

      - guard: .state == spacewar:torpedo
        . = spacewar:torpedo_update(., consts)

      - guard: .state == spacewar:exploding
        (., prng) = spacewar:explosion_tick(., prng)

      - guard: .state == spacewar:hyperspace_in
        (., prng) = spacewar:hyperspace_transit(., prng, consts)

      - guard: .state == spacewar:hyperspace_out
        (., prng) = spacewar:hyperspace_breakout(., prng, consts)

      - guard: .state == spacewar:empty
        pass

      - else:
        pass
```

Semantics, precisely:

1. The dispatch folds over the collection **in index order**, 0 to len−1. This is
   not an authored schedule; it is the definition of a fold over an ordered
   collection — the same way a DIGS `for` visits a list. Where elements are
   independent, the order is unobservable; where they share a thread, the thread
   makes the order a derived fact.
2. For each element, the guards are evaluated and **exactly one arc fires**
   (§Guards). Within the arcs, `.` is the current element and `slot` is its index.
3. The firing arc's step runs: `.` rebinds to the element's new version; threaded
   names rebind hand-to-hand, carrying out of element *i* into element *i+1*.
4. `pass` is the explicit identity arc: the element and threads pass through
   unchanged.
5. The dispatch's result is the tuple (new collection, final threaded versions).

The `threading (...)` clause names the places passed hand-to-hand through the fold.
It is how every classic "shared mutable resource" is expressed purely: the PRNG, a
spawn queue, a score accumulator. A dispatch with an empty threading set is a pure
per-element map, and a runtime may execute its elements in parallel.

An arc target may also receive the *whole* collection as a read-only argument
(`spacewar:perception(., objects, consts)`) for cross-entity queries. It reads the
**input version** of the collection — the version the dispatch was given — so the
fold's per-element writes never feed back into the same pass. A cross-entity effect
that must observe earlier elements' updates within the same pass is data, and is
expressed by threading it.

### Iterate step

Bounded repetition of a sub-wiring — a fold whose body is wiring:

```
- bodies = iterate 8 threading (bodies):
    wiring:
      - bodies = physics:relax_constraints(bodies, params)
```

The bound is an integer literal or a field path read once, before the first
iteration; `n <= 0` makes the step an identity. The threaded places pass hand-to-hand
through the iterations. Iterate steps may nest — totality is structural
(fixed-before-loop bounds), not a static ceiling.

There is no `iterate until converged`. Convergence-to-fixpoint has no
fixed-before-loop bound, so it has no place inside a tick; the iteration count is a
declared game-design fact (more iterations, more accuracy, more cost). A game that
wants convergence across time carries the residual in a `state:` place and continues
next tick.

### Guards and the two laws

A guard is a DIGS boolean expression (the DIGS expression grammar, reused) over the
current element's Fields, `slot`, and in-scope places. Guard expressions are DIGS
**everywhere in the wiring** — including arcs that route to Processors whose bodies
are target-native platform code. Wiring is wiring; only bodies vary.

Every guard set — a dispatch's arcs, or a guarded step with its implicit else —
must satisfy two laws:

1. **Totality (the determinism law).** The guards form a **complete partition**:
   for every possible element, **exactly one** guard is true — no overlaps, no
   gaps. When the discriminant is a **closed enum**, a tool verifies exhaustiveness
   and exclusivity by enumeration, and a missing variant is a compile-time error.
   When any guard term is open-typed (an integer, a compound condition like
   `slot == 0`), completeness is unprovable, so an explicit **`else:` arc is
   mandatory**. The Spacewar dispatch above shows both: the enum variants are
   covered exhaustively, and because the `ship` arcs also test `slot`, the `else`
   arc is required to make the partition total. `pass` and `else` exist precisely
   so that "do nothing" and "everything not otherwise covered" are explicit,
   checkable wiring instead of silent fall-through.
2. **Thinness (the composability law).** A guard **reads a fact; it never computes
   one.** It tests Fields a Processor already produced — a state tag,
   `can_see_player` — with no sub-Processor calls, no new values, no work. If a
   branch needs a fact that doesn't exist yet, a Processor computes it into a Field
   first, and the guard reads that: **perceive → route → act**.

The division of labor: decision structure lives in the arcs; computation lives in
Processors. A guard never computes; a Processor never routes. **No Processor
contains an `if` that steers the Network** — every arc target is
state-machine-ignorant and composable, which is why a state machine in RUNS is a
Network of guarded arcs, never a Processor.

That law is **decomposition-relative**. A coarse Processor may legally hold an
internal `if state == torpedo:` dispatch — from the Network's view it is one opaque
transition, so there is no Network routing to steer (a legitimate
mid-decomposition station; granularity is a spectrum, and no point on it is
non-compliant). The law binds once the routing targets are distinct Processors:
"which Processor runs" is then a wiring decision, and resurrecting it as a
dispatcher Processor is the forbidden steering. A body `if` that *computes* a
discriminant (sets `state = exploding`) is action, not steering — producing a tag
is a Processor's job; only reading one to route is a guard's.

Guards and preconditions are the same fact stated on the two sides of the seam: the
guard *asserts* it on the arc, the Processor's precondition *assumes* it in the
body. In a well-formed Network the guards discharge every precondition (see
[DIGS §Preconditions](./DIGS_EXPRESSION_LANGUAGE.md#preconditions)).

---

## The Platform Boundary

### Boundary Records, and Records only

The `requires:` and `produces:` blocks are the **boundary Records** — the entire
contract between the game and the platform:

- **Inbound** (`requires:`): the host writes them before each tick — timing, player
  input, transport-delivered remote input.
- **Outbound** (`produces:`): the host reads them after each tick — render state,
  audio triggers, match results.

The boundary is **Records only — never Processors**, and necessarily so. It joins
two paradigms: deterministic, platform-agnostic computation on one side and
platform-coupled machinery on the other. A noun is paradigm-neutral and can sit on
the seam; a verb always carries a paradigm, so a "boundary Processor" would have to
pick a side — at which point it is not the boundary but the first Processor of
whichever side it joined. Any shape-translation is a Processor on one side *up to*
the boundary, never *in* it.

### Game-authored platform Processors vs. the host

Two different things live beyond a deterministic region's edge, and they are not
the same:

- **Game-authored platform Processors** — the game's own I/O logic: building the
  render representation for a target, mapping device input to intent, driving
  audio. They are verb-pure Processors and full citizens of the Network, but their
  bodies are written in **whatever the target demands** — the standard deliberately
  specifies no language for them, because they are defined to be rewritten per
  target. They are baked into the Build.
- **The host (runtime)** — the generic, game-agnostic program that runs Builds: a
  browser, a console, a libretro-style core, a server. It exposes platform
  capabilities, writes the inbound boundary Records, and reads the outbound ones.

Where the line falls between them is platform-dependent: a fantasy console exposing
`draw_sprite` leaves the game little platform code to author; a raw framebuffer
leaves it all. Both arrangements are conforming; the boundary Records are the
contract in every case. The conventions layer names the regions this split creates
and the Variant/Port taxonomy it enables — see the
[RUNS Library](https://github.com/enduring-game-standard/runs-library).

### Time, deterministically

A deterministic region intended to replay identically — for lockstep multiplayer,
replays, or verification — must receive time as data with one of two shapes: a
**constant tick quantum** (a fixed timestep baked into the game's constants) or a
**tick counter**. A wall-clock `delta_time` measured by the host varies per run and
destroys replay determinism for the entire region downstream of it. Variable-rate
concerns — render interpolation, substepping against real elapsed time — belong on
the platform side of the boundary. The recommended boundary shapes for this live in
the RUNS Library.

---

## Verification Properties

A conforming validator checks the following statically. Violations are errors.

1. **Bipartite invariant.** Every data dependency passes through a place. A value
   produced by one Processor and read by another lands in a Record Field between
   them, in the wiring.
2. **Single assignment.** After lowering name reuse to explicit versions, no place
   version is written twice, and every reference resolves to exactly one version.
   "Two writers, same tick, no thread between them" is the canonical failure.
3. **Guard partition.** Every guard set is exclusive and exhaustive: closed-enum
   discriminants checked by enumeration; any open term forces an explicit `else`.
   Overlapping guards are an error, not a warning — under a true partition there is
   nothing left for execution order to decide.
4. **Guard thinness.** Guards are pure reads of in-scope Fields: no sub-Processor
   calls, no construction, no arithmetic beyond comparison and boolean composition
   of read values.
5. **Tick acyclicity.** The version-resolved dataflow graph of one tick is a DAG.
   (With single assignment this holds by construction; the check is that name
   resolution found no forward references.)
6. **Binding completeness.** Every invocation binds all declared outputs; every
   `produces:` place and every Sub-Network `outputs:` place is assigned exactly
   once per tick on every path (guarded steps count as assignment on both arms).
7. **Type agreement.** Every argument and binding matches the target's declared
   types exactly.
8. **Precondition discharge.** For every arc into a Processor, the guard (plus path
   facts) implies the Processor's declared preconditions, where implication is
   decidable; where it is not, the validator reports the obligation so a test
   vector can cover it.

A validator does **not** check granularity ("too coarse" does not exist), does not
require list bounds, and does not compute a mandatory static firing ceiling. A
worst-case execution or memory report over declared bounds is a useful *advisory*
for target-fitting — never a compliance gate.

---

## Formal Grammar

EBNF. Terminals in double quotes; `{ }` is zero-or-more; `[ ]` is optional. INDENT
denotes one two-space level relative to the enclosing line. Lexical rules (encoding,
comments, identifiers, qualified names, literals) are shared with
[DIGS](./DIGS_EXPRESSION_LANGUAGE.md#lexical-structure).

```ebnf
network_file      = network_decl ;

network_decl      = "network" qualified_name NEWLINE
                    ( root_blocks | sub_blocks )
                    wiring_block ;

root_blocks       = requires_block produces_block [ state_block ] [ local_block ] ;
sub_blocks        = inputs_block outputs_block ;

requires_block    = INDENT "requires:" NEWLINE { INDENT INDENT place_decl NEWLINE } ;
produces_block    = INDENT "produces:" NEWLINE { INDENT INDENT place_decl NEWLINE } ;
state_block       = INDENT "state:"    NEWLINE { INDENT INDENT place_decl [ source_annot ] NEWLINE } ;
local_block       = INDENT "local:"    NEWLINE { INDENT INDENT place_decl NEWLINE } ;
inputs_block      = INDENT "inputs:"   NEWLINE { INDENT INDENT place_decl NEWLINE } ;
outputs_block     = INDENT "outputs:"  NEWLINE { INDENT INDENT place_decl NEWLINE } ;

place_decl        = identifier ":" qualified_name [ collection_spec ] ;
collection_spec   = "[" integer_literal "]"
                  | "[" "max:" integer_literal "]"
                  | "[" "]" ;

source_annot      = NEWLINE INDENT INDENT INDENT "source:" qualified_ref ;
qualified_ref     = qualified_name { "/" identifier } ;

wiring_block      = INDENT "wiring:" NEWLINE { INDENT INDENT step } ;

step              = "-" ( invocation_step
                        | guarded_step
                        | dispatch_step
                        | iterate_step ) ;

invocation_step   = binding "=" invocation NEWLINE ;
binding           = identifier
                  | "(" identifier { "," identifier } ")" ;

guarded_step      = "guard:" expression NEWLINE
                    INDENT invocation_step ;

dispatch_step     = binding "=" "dispatch" identifier
                    [ "threading" "(" identifier { "," identifier } ")" ]
                    ":" NEWLINE
                    INDENT "arcs:" NEWLINE
                    { INDENT INDENT arc } ;

arc               = "-" ( "guard:" expression | "else:" ) NEWLINE
                    INDENT ( arc_step | "pass" NEWLINE ) ;
arc_step          = arc_binding "=" invocation NEWLINE ;
arc_binding       = arc_name
                  | "(" arc_name { "," arc_name } ")" ;
arc_name          = "." | identifier ;

iterate_step      = binding "=" "iterate" bound
                    [ "threading" "(" identifier { "," identifier } ")" ]
                    ":" NEWLINE
                    INDENT wiring_block ;
bound             = integer_literal | field_path ;

invocation        = qualified_name "(" [ arg_list ] ")" ;
arg_list          = argument { "," argument } ;
argument          = "." | field_path ;

(* expression: the DIGS expression grammar; "." and "slot" are in scope
   inside dispatch arcs *)
field_path        = identifier { "." identifier } ;
qualified_name    = [ identifier ":" ] identifier ;
integer_literal   = DIGIT+ ;
```

---

## Deliberate Exclusions

The wiring layer does not and will never express:

| Excluded | Rationale |
|----------|-----------|
| Authored execution order (`phases:`, priorities, scheduling hints) | Order is derived from dataflow. A Network that "needs" authored order has an unthreaded shared resource; the fix is threading, not a primitive. |
| Convergence loops (`iterate until stable`) | No fixed-before-loop bound, so no place inside a tick. Iteration counts are declared design facts; convergence across time uses `state:`. |
| Intra-tick feedback (cyclic data dependencies) | One tick is a DAG by single assignment. Feedback is the cross-tick loop through `state:` places — the one unbounded loop, visible in the topology. |
| Dynamic topology | Processors and arcs are not added, removed, or rewired at runtime. Variation at runtime is guard conditions over static arcs; variation across builds is editing the wiring and baking again. |
| Event-driven execution | A Network is a synchronous computation fired once per tick. Event collection and buffering are the host's business; inbound boundary Records are the only door in. |
| Side effects | The Network produces outbound boundary Records; what becomes of them — pixels, waveforms, packets — happens beyond the boundary. |

---

## Versioning

Network files carry no per-file version declaration in version 1.0; the `network`
keyword identifies the file kind. Evolution rules match DIGS: additive only; a
Network valid under 1.0 is valid and means the same thing forever; no feature
flags. Published Networks are content-addressed, so any revision is a new artifact
with a new ID.

---

## The Oracle: Reference Tooling and Test Vectors

The wiring layer's claims are **asserted** until a reference implementation checks
them. The minimum tooling:

1. **Reference parser** — `.runs` Network text to a structural representation
   (places, steps, arcs, guards).
2. **Reference validator** — the checks in §Verification Properties, including the
   SSA lowering of name reuse.
3. **Reference scheduler** — from a parsed Network and initial place values,
   produce one tick's execution as an explicit sequence of invocations with
   resolved versions. The scheduler is the oracle: any conforming runtime's tick,
   in any execution order it chooses, must produce place values identical to the
   scheduler's.
4. **Tick-level test vectors** — known initial state in, expected final state out,
   for the first converted game: two ships and no torpedoes; a ship firing
   (spawn-request thread exercised); a ship exploding; the PRNG thread consumed in
   slot order.

---

*MIT License*
