# RUNS Conversion Process

*How to convert a game from original source to RUNS source.*

> **Status**: This process has been executed once (Spacewar! 3.1, PDP-1 → RUNS).
> Its failure-mode tables are derived from that conversion's two postmortems. The
> resulting Spacewar source is **asserted** faithful — verified by no oracle yet —
> and the process below states at every step which claims static work can earn and
> which only an oracle can.

## What a Conversion Is

A conversion re-expresses an existing game's logic as RUNS source — Records,
Processors, and Networks — such that a build baked from that source behaves
**bit-identically** to the original. The bar is exact: the goal is the original
game's rules, including its overflow behavior, its iteration order, its
representation quirks. A re-expression that is merely "close" or "improved" is not
a conversion of the game; it is a **variant** of it, and must say so.

Fidelity claims come in exactly two strengths, and every document produced by this
process must use the right word:

- **Asserted** — a person or AI read the source and judged the translation
  faithful. This is the strongest claim any amount of reading can earn.
- **Verified** — output checked mechanically against an **oracle**: test vectors
  derived from the original platform's behavior (typically an emulator), evaluated
  by a reference evaluator, matching bit-for-bit.

The process is seven steps, strictly ordered. Each step's inputs are produced by
prior steps; each produces durable artifacts, so work can pause at any boundary and
resume under a different person or agent without loss of context.

---

## Step 1: Source Acquisition and Deep Reading

### Definition

Obtain the canonical source and read every line — not skim, read. Produce a
machine-checked concordance mapping every line of original source to a functional
description.

### Outputs

1. **Canonical source archive** — the definitive version, committed. If multiple
   versions exist, the choice is documented with reasoning.
2. **Source concordance** — a line-by-line (or routine-by-routine) mapping:
   `line range → routine → what it does → which RUNS artifact will consume it`.
   Every line accounted for.
3. **Edge-case inventory** — every non-obvious behavior found through source
   reading, play, and external research. Each entry: `behavior → source evidence →
   why it matters`.

### How

- Write a parser for every structured data block in the source (tables, constant
  blocks, data arrays). Never count by hand. The parser is authoritative over any
  analysis document — including this process's own outputs.
- Play the original on real hardware or an accurate emulator. Observe what the code
  doesn't explain.
- Read the external record: designer notes, interviews, community analyses.
- For each magic number, determine its derivation. A bitmask? A maximum? A
  sentinel? Document the *why*, not just the value.

### Acceptance Criteria

- Every line of original source appears in the concordance exactly once.
- No magic number remains underived.
- The edge-case inventory contains at least one entry that reading alone did not
  surface (found through play or research).

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Incomplete reading | Missing Processors in later steps | The concordance enforces completeness |
| Manual data counts | Wrong numbers propagate (Spacewar: an analysis doc said 478 stars; the parser found 469) | Parse mechanically. Always. |
| Missing edge cases | Bugs discovered only at first run | Play the game. Ask "what happens when this overflows?" |

---

## Step 2: The Boundary Contract

### Definition

Before defining any Record or Processor, draw the three-way boundary: what is
**game logic** (RUNS source, the deterministic region), what is **entity identity**
(AEMS), and what is **platform contact** (everything beyond the boundary Records).
Write the boundary contract first — it determines the shape of everything
downstream, and the first conversion learned this by doing it last.

### Outputs

1. **Boundary Record contract** — the inbound (`requires:`) and outbound
   (`produces:`) Records, every Field typed. Inbound: timing (as a constant tick
   quantum or tick counter — never wall-clock; see
   [Network Topology §Time](./NETWORK_TOPOLOGY.md#time-deterministically)), player
   controls. Outbound: render state, audio triggers, match results. This contract
   is the entire interface between the converted rules and any platform, forever.
2. **AEMS layer document** — every candidate entity run through the AEMS four-test
   rubric; each accepted (Entity/Manifestation classification) or rejected (with
   reasoning). Exclusions matter as much as inclusions.
3. **Platform-contact inventory** — what the original program did that is not game
   logic: display timing, device I/O, sound generation. Each item lands either in
   the host's duties or in game-authored platform Processors, but never inside the
   deterministic region.

### How

- The boundary rule, tested and held by the first conversion: **if changing a value
  changes how the game plays, it is RUNS; if it changes only what something looks
  or sounds like, it is not.** A simulation whose output feeds back into Fields any
  game-logic Processor reads is game logic, however "physical" it looks.
- Expect the original to interleave logic and rendering — 1960s–1990s code draws
  mid-loop because the hardware demanded it. Separating them is valid exactly when
  rendering writes nothing game logic reads; prove that from the concordance, then
  separate without fear.
- Write the contract before Records. Every later step fills in one side of it.

### Acceptance Criteria

- Every game noun evaluated against the four-test rubric; at least one candidate
  excluded with documented reasoning.
- The boundary contract specifies every inbound and outbound Field with types.
- Time enters as a tick quantum or counter, not a measured delta.
- Every platform behavior in the original is assigned a side of the boundary.

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Boundary as afterthought | Platform concerns discovered ad hoc during Processor writing | Contract first. The first conversion's postmortem is unambiguous on this. |
| Logic and rendering left fused | The deterministic region depends on display state | Prove rendering writes nothing logic reads, then cut at the Record. |
| Entity over-identification | Bloated AEMS layer, tangled Records | The rubric. A mechanics-first game can be three entities deep and that is the correct answer. |

---

## Step 3: Type System and Record Schemas

### Definition

Define every piece of game state as typed Fields in named Records
([Record Schema](./RECORD_SCHEMA.md)). Declare game-defined numeric types that pin
the original platform's arithmetic. Extract and derive every constant.

### Outputs

1. **Type declarations** — every game-defined type with its full contract: storage,
   width, complement, binary point, range
   ([DIGS §Game-Defined Types](./DIGS_EXPRESSION_LANGUAGE.md#game-defined-types)).
   If the original machine is ones-complement, say so in the type and let the
   declared semantics carry it — including the two zeros, if the original code
   exploits them.
2. **Record schemas** — every Record, every Field typed, every Field traced to the
   concordance.
3. **Constants** — every magic number named, typed, and documented with its
   derivation from source.

### How

- Start from the original's state: every variable, register convention, and table
  slot must map to a Field. Cross-reference against the concordance.
- Where the original moves a binary point between contexts (sin/cos arguments vs.
  positions vs. sqrt results), document the convention **per context**, not just
  per type — and keep the moves as explicit shifts in bodies, exactly as the
  original wrote them.
- Declare collection bounds that the original guarantees (`spacewar:object[24]`).
  The bound is part of the game's rules; changing it later to fit a smaller target
  is a rules change, hence a variant.

### Acceptance Criteria

- Every piece of mutable game state is a typed Field on a Record.
- Every constant has a documented derivation.
- Game-defined types pin every representation edge: overflow, complement, negative
  zero, rounding.
- No Field is untyped; no source variable is unmapped.

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Wrong numeric semantics | Plausible-but-wrong math (the first conversion's sqrt scaling bug class) | Pin binary-point conventions per context; test vectors in Step 5 catch the rest |
| Missing Fields | Processors need state that has no home | The concordance cross-reference is mandatory |
| Under-specified types | Two implementations disagree at the edges | Define every edge: max value, zero, negative zero |

---

## Step 4: Processor Extraction and Network Wiring

### Definition

Identify every discrete operation in the original; declare each as a Processor with
typed inputs, outputs, and preconditions. Wire them into Networks per
[Network Topology](./NETWORK_TOPOLOGY.md).

### Outputs

1. **Processor signatures** — name, inputs, outputs, preconditions for every
   Processor.
2. **Network wiring** — the root tick Network and any Sub-Networks, with guards,
   threading, and the boundary blocks from Step 2.
3. **Dataflow trace** — for each wire, which source lines justify it.

### How

- Walk the original's main loop. Each distinct operation becomes a Processor.
  **Granularity is a spectrum and every point is compliant**: a whole foreign
  routine re-expressed verb-for-verb as one Processor is a legitimate Processor.
  Convert first; decompose later, if ever. Do not let factoring ambitions delay or
  distort fidelity.
- **Translate the structure, not the machine.** The original is full of its own
  platform's workarounds — register overwriting, in-place tables, hand-scheduled
  ordering. Those are the *source machine's* realization details, not game rules.
  In RUNS they become threads: the PRNG every routine touches is a threaded chain;
  the table updated in place is a fold producing its next version; "routine A runs
  before routine B" becomes a wire where B reads what A wrote. If the wiring seems
  to need an authored order the data doesn't imply, an unthreaded resource is
  hiding; find it and thread it.
- Where the original processes entities in a fixed order with observable
  consequences (ship 1 fires before ship 2 sees the world), the dispatch fold in
  index order plus the threads reproduces it exactly — that order is derived from
  the wiring, and conforming runtimes cannot disagree about it.
- Route by state with guards forming a complete partition: closed-enum
  discriminants wherever the original had a state machine; an explicit `else` arc
  wherever any guard term is open. Trace the original's conditional dispatch
  exactly.
- Declare a precondition for every assumption a body makes, and check that the
  guards routing into it discharge each one.

### Acceptance Criteria

- Every source routine maps to exactly one Processor (or is documented as
  excluded, with reasoning).
- Every Processor input and output is a Field defined in Step 3.
- All shared resources are threaded; the wiring contains no authored ordering.
- Every guard set partitions; every precondition is discharged by its arcs.
- The Network validates against
  [Network Topology §Verification Properties](./NETWORK_TOPOLOGY.md#verification-properties).

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Phantom dataflow | A Processor "returns" data that lands in no Record; it cannot be inspected, persisted, or rewired | Every inter-Processor value is a Field on a place in the wiring — the bipartite invariant |
| Imported machine artifacts | Place overwrites and ordering directives copied from the source machine | Thread shared things; order derives. Overwrite and authored order are bug-smells, not features. |
| Missing guard conditions | A Processor fires for entity states the original never ran it on | Trace the original dispatch; declare preconditions; let the validator check discharge |
| Stale reads | A Processor reads a pre-update version its original read post-update | The dataflow trace catches it: each wire cites the source lines it reproduces |

---

## Step 5: Processor Bodies and Test Vectors

### Definition

Write every Processor's algorithm in DIGS. Create test vectors **alongside** each
body — not after.

### Outputs

1. **Processor bodies** — complete `.runs-prim` files. Every source line the
   concordance assigns to a Processor is represented in its body.
2. **Test vectors** — for every Processor that computes: `{inputs →
   expected_outputs}` pairs derived from (a) hand-traced executions of the original
   source, (b) observed original behavior, (c) boundary values of the type system.
   Minimum per math Processor: zero, maximum, minimum, one typical, one boundary
   case.

### How

- **Translate literally.** Do not improve. Do not optimize. Do not reinterpret. If
  the source corrects an angle with `if angle > pi: angle -= pi`, write exactly
  that — not the "mathematically nicer" full-circle wrap. The first conversion's
  hand-compilation introduced every one of its bugs by improving things; the
  literal translations were the correct ones. Fidelity is correctness, and any
  deviation is documented as intentional, with reasoning, or it is a defect.
- The body is the specification. A header plus prose comments is a promise without
  proof; the executable body is what makes mechanical compilation produce the right
  binary-point behavior without anyone needing to understand *why* the algorithm
  scales what it scales.
- If you cannot write a test vector for a body, you do not yet understand the
  algorithm. Stop and trace the original until you can.

### Acceptance Criteria

- Every Processor has a complete DIGS body that parses against the grammar.
- Every computing Processor has vectors covering zero / max / min / typical /
  boundary.
- Every concordance line is consumed by exactly one body (or documented exclusion).
- No body "improves" on the original; all intentional deviations are listed with
  reasoning.

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| "Improving" the algorithm | Behavior diverges from the original at runtime | Translate literally; the original is scripture, commentary goes in comments |
| Vectors written after | Bugs surface at first run instead of at writing time | Vectors alongside bodies, no exceptions |
| Wrong binary-point handling | Plausible but wrong math | `sqrt(2048) → 23168`, not 45: a single vector catches the whole class |

---

## Step 6: Static Verification

### Definition

Check everything that can be checked without executing: structural completeness,
wiring validity, concordance coverage. State plainly what static work cannot prove.

### Outputs

1. **Verification report** — concordance coverage, validator results
   (single-assignment, partitions, preconditions, binding completeness), vector
   inventory, known deviations.
2. **Confidence statement** — an honest declaration: which claims are now
   *asserted*, and that none are *verified* until Step 7's oracle runs.

### How

- Mechanically: every source line → exactly one body or a documented exclusion.
- Run the Network validator's checks (or perform them by hand against
  [Network Topology §Verification Properties](./NETWORK_TOPOLOGY.md#verification-properties),
  documenting each).
- For every entity state in every enum, confirm some arc handles it.
- Hand-trace the critical math bodies against their vectors if no evaluator exists
  yet — and record that a hand-trace is itself an assertion, not verification.

### Acceptance Criteria

- 100% concordance coverage.
- Validator checks pass (or each manual check is documented).
- The confidence statement uses *asserted*/*verified* correctly — no
  verified-tier word ("bit-exact", "faithful", "validated") appears for any claim
  no oracle has checked.

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| False confidence from static checks | "Verified" appears in documents describing unexecuted source | The two-tier vocabulary is mandatory; static work caps out at *asserted* |
| Missing state coverage | An entity state no arc handles | Enumerate states × guards mechanically |

---

## Step 7: First Build and the Oracle

### Definition

Produce a playable Build for one target, and stand up the oracle that converts
*asserted* into *verified*: a reference evaluator running the test vectors against
ground truth from the original platform.

### Outputs

1. **A playable Build** on at least one target.
2. **Oracle results** — per-Processor vector outcomes against the reference
   evaluator and against original-platform (emulator-derived) ground truth. Each
   Processor's status flips to *verified* only when its vectors pass bit-for-bit.
3. **Deviation manifest** — if the Build diverges from strict evaluation anywhere
   (substituted sqrt, native floats, re-encoded data tables), every divergence
   recorded per
   [DIGS §Divergent Compilation](./DIGS_EXPRESSION_LANGUAGE.md#divergent-compilation),
   and the Build labeled honestly as a **variant**.
4. **Postmortem** — what the Build revealed about the source, the spec, and this
   process. A deliverable, not an option.

### How

- Choose the simplest viable target. Run vectors **before** playing — fix math
  before judging feel.
- Know which of two things you are doing, and label it. **Machine compilation**
  parses and compiles the RUNS source mechanically. **Hand-compilation** is a
  person or AI reading the source and emitting target code by hand. A
  hand-compilation can demonstrate playability and surface spec gaps; it verifies
  nothing, and its output is not evidence of source correctness — the first
  conversion's hand-compilation introduced four bugs into a correct source, every
  one a deviation from literal translation.
- Expect data-representation work: original data encodings can exceed a target's
  numeric range (Spacewar's 18-bit outline words overflow a 16-bit console's
  integers). Re-encoding data at bake time is legitimate compilation; record it in
  the deviation manifest if and only if the decoded values differ.
- Compare against the original running in an emulator. Same inputs, same seeds,
  same outcomes.

### Acceptance Criteria

- The Build runs and is playable.
- All vectors pass on the Build's compiled bodies — or every failure is either
  fixed or covered by a manifest entry.
- Every deviation from strict evaluation is in the manifest; a divergent Build is
  labeled a variant.
- The per-Processor status table (asserted/verified) is published with the source.
- The postmortem exists.

### Known Failure Modes

| Failure | How it manifests | Prevention |
|---------|-----------------|------------|
| Translation errors in the build step | Gameplay defects with a correct source | Run vectors first; prefer machine compilation as soon as tooling exists |
| Improvement during emission | Reinterpretation bugs | Literal translation applies to compilation too |
| Silent divergence | A "port" that is actually an unlabeled variant | The deviation manifest is mandatory; divergence makes a variant, and the label is the honesty |
| Skipping the postmortem | The next conversion repeats this one's mistakes | It is a deliverable with acceptance criteria, like everything else |

---

## The Deliverables, Together

A finished conversion publishes:

| Artifact | Step | Claim it carries |
|----------|------|------------------|
| Canonical source archive + concordance + edge cases | 1 | — |
| Boundary contract, AEMS layer, platform inventory | 2 | — |
| Type declarations, Record schemas, constants | 3 | Asserted |
| Processor signatures + Network wiring | 4 | Asserted, validator-checked |
| DIGS bodies + test vectors | 5 | Asserted |
| Verification report + confidence statement | 6 | Asserted (explicitly) |
| Build + oracle results + deviation manifest + postmortem | 7 | Verified, per Processor, as vectors pass |

The end state to aim at: RUNS source complete enough that an implementer — human,
agent, or compiler — with **no access to the original source** can bake a Build
whose behavior matches the original bit-for-bit under strict evaluation, and an
oracle that proves it Processor by Processor.

---

*MIT License*
