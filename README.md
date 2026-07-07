# RUNS: Records Update on Neutral Substrate

🏠 **[EGS Overview](https://github.com/enduring-game-standard)** · 📦 **[AEMS](https://github.com/enduring-game-standard/aems-schema)** · 🎯 **[AEMS Conventions](https://github.com/enduring-game-standard/aems-conventions)** · 🔧 **[RUNS](https://github.com/enduring-game-standard/runs-spec)** · 📖 **[RUNS Library](https://github.com/enduring-game-standard/runs-library)** · ⚡ **[WOCS](https://github.com/enduring-game-standard/wocs-protocol)** · 🎼 **[MAPS](https://github.com/enduring-game-standard/maps-notation)** · 🎶 **[MAPS Library](https://github.com/enduring-game-standard/maps-library)** · ❓ **[FAQ](https://github.com/enduring-game-standard/.github/blob/main/profile/FAQ.md)** · 🔤 **[Glossary](https://github.com/enduring-game-standard/.github/blob/main/profile/README.md#glossary)**

> **Status**: Design stage. No working reference implementation exists yet — no parser,
> no evaluator, no compiler. The specifications below are precise enough to implement
> from, and the first implementation is expected to correct them. Every fidelity claim
> in this ecosystem is currently **asserted** (a person or AI judged it correct by
> reading), not **verified** (checked mechanically against an oracle). The documents
> say which is which. Ecosystem-wide state and the verification milestone:
> [STATUS.md](https://github.com/enduring-game-standard/.github/blob/main/profile/STATUS.md).

## What RUNS Is

**RUNS** (Records Update on Neutral Substrate) is a plain-text source format for game
logic. The substrate is the source medium itself: it binds to no engine, no vendor, no
renderer, and no hardware generation. Game logic written as RUNS source is compiled —
*baked* — into platform-specific builds; the source endures while platforms churn.

RUNS applies the Linux model to game execution. There is no "RUNS engine," the way
there is no single "Linux OS": each game assembles its own engine from shared
components into a standalone build, the way Android and the Steam Deck are different
operating systems built around the same kernel and coreutils. An open-source engine
does not provide this — it removes the vendor but keeps the coupling, since the game
is still written inside one engine's runtime and version churn. RUNS removes the
coupling: the rules are source that outlives every engine that realizes them. This is
a design-intent claim about the specification below, checkable by reading it — not a
status claim about adoption.

RUNS has exactly four primitives and one sub-language:

| Piece | Kind | What it is |
|-------|------|------------|
| **Record** | noun | A named collection of typed Fields — a data store. The only thing data flows through. |
| **Field** | data | A typed, named value on a Record. The type system is open. |
| **Processor** | verb | A transformation that reads Fields and writes Fields, holding no state of its own. |
| **Network** | wiring | The explicit bipartite graph wiring Records to Processors. Defines one tick. |
| **DIGS** | sub-language | Deterministic Inspectable Game Syntax — the pure, total, deterministic language Processor bodies are written in. |

These map to three layers: **data** (Records and Fields), **wiring** (Networks), and
**computation** (Processor bodies, written in DIGS). DIGS is a named sub-language of
RUNS — not a separate protocol. It is governed and versioned here.

Compliance is the four primitives plus DIGS — nothing more. The `runs:` namespace of
recommended shared shapes is **optional**; a game built entirely on its own prefix
(`spacewar:`, `pong:`) is fully compliant. The shared palette lives in the
[RUNS Library](https://github.com/enduring-game-standard/runs-library), the
conventions layer of the ecosystem.

## The Documents

Each document is self-sufficient and owns one layer:

| Document | Owns |
|----------|------|
| [Record Schema](./RECORD_SCHEMA.md) | The data layer: Record, Field, enum, and game-defined type declarations; publication and identity; initial data sources. |
| [DIGS Expression Language](./DIGS_EXPRESSION_LANGUAGE.md) | The computation layer: the full language of Processor bodies — grammar, type semantics, evaluation rules, the strict-evaluation contract, and divergent compilation. |
| [Network Topology](./NETWORK_TOPOLOGY.md) | The wiring layer: how Records and Processors compose into a tick — single-assignment dataflow, guarded dispatch, derived execution order, and the boundary contract with the platform. |

Together these three documents fully specify RUNS — the data, computation, and wiring
layers. A reader implementing RUNS from scratch needs all three.

## How a Game Is Built and Distributed

Every RUNS artifact — each Record schema, enum, type, Processor, and Network — is an
independently published, content-addressed event on the commons (Nostr), referenced by
ID. A Network references the Records and Processors it wires by ID and never embeds
them; it stays pure wiring. A manifest pins a game's exact dependency closure; a build
tool resolves that closure, flattens the graph, and bakes it into a self-contained
**Build** for one target — a native binary, a wasm bundle, a cartridge.

The commons is touched only at lifecycle boundaries: discovery, build-time dependency
resolution, and load/save of persistent player data (AEMS). The gameplay tick loop has
no dependency on Nostr, relays, or network connectivity. The same relationship holds
between any package registry and the programs built from it.

Two facts follow from publication by ID. First, the full source of any game is findable
and studyable on the commons. Second, anyone can open a game's Network, swap Processors,
and bake a **variant** — variation is first-class and permissionless, the way physical
games have always accrued house rules.

## Status, Honestly

What exists today:

- These specifications, written to be implementable in isolation.
- A complete conversion of Spacewar! 3.1 (1962, PDP-1) into RUNS source
  (RUNS-Spacewar, not yet published): 26
  Processors, asserted faithful by line-level source tracing. An AI has hand-compiled
  that source into a playable PICO-8 cartridge. A hand-compilation demonstrates
  playability; it verifies nothing.

What does not exist yet:

- A reference parser, evaluator, or compiler.
- Test vectors checked against original-platform behavior (the oracle that would turn
  *asserted* into *verified*).
- A second game, or a second platform reached by machine compilation.

The base specification is closed: it changes when its author changes it. There is no
proposal process here, and none is implied. The open door for contribution is the
conventions layer — shared shapes and signatures in the
[RUNS Library](https://github.com/enduring-game-standard/runs-library).

**MIT License**
