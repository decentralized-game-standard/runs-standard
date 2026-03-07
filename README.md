# RUNS: Record Update Network System

🏠 **[EGS Overview](https://github.com/enduring-game-standard)**  
· 📦 **[AEMS](https://github.com/enduring-game-standard/aems-schema)**  
· ⚡ **[WOCS](https://github.com/enduring-game-standard/wocs-protocol)**  
· 🎼 **[MAPS](https://github.com/enduring-game-standard/maps-notation)**  
· ❓ **[FAQ](https://github.com/enduring-game-standard/.github/blob/main/profile/FAQ.md)**

> **Status**: Draft / RFC  
> **Version**: 0.1.0

## A Composable Substrate for Enduring Games

**RUNS** (Record Update Network System) is a composable, plain-text source format for game logic. Stateless Processors transform Records through explicit Networks, giving games a neutral substrate designed to outlive any single engine, company, or hardware cycle. RUNS source is compiled into platform-specific binaries by runtimes — the source endures on the commons; binaries are rebuilt for each new platform.

RUNS treats game engines the way Unix treats operating systems: as pipelines of small, focused components wired together. Records hold state. Processors transform it. Networks define the wiring. Each piece evolves independently. Variation is native: anyone can open a Network, add or replace Processors, and compile a new variant — the same way anyone can play soccer with house rules. Obsolescence in one layer never cascades to another. Diverse runtimes, from minimal web builds to high-performance native, share the same ecosystem.

RUNS components — Networks, Records, Processors, and ecosystem packages — are plain-text Nostr events by convention. This is not a deployment detail. Nostr is the commons: the mechanism by which composable game components become discoverable, inheritable, and remixable across generations. Provenance chains through note IDs ensure self-describing data that any future community can find, study, and build upon without permission.

RUNS is not a full engine or rule language. It is the minimal kernel layer: a neutral substrate for transforming data, with game-specific logic implemented in swappable Processors and coordinated across the broader Enduring Game Standard (EGS).

## Why RUNS?

Current engines are monolithic and fragile:

- Inheritance of one company's physics, rendering, and fate.
- Variation limited to restricted scripting APIs — if allowed at all.
- Game logic locked inside opaque binaries that no one outside the studio can read, maintain, or learn from.

RUNS unbundles everything:

- **Natural Variation** — Anyone can open a Network, swap Processors, extend Records, and compile a variant. This is not modding — it is the composability that physical games have always had.
- **Portability** — Logic runs on any compliant runtime (web, desktop, embedded).
- **Interoperability** — Shared data shapes let Processors from different developers compose freely.
- **Open Commons** — Plain-text Nostr events place components in an open commons where any developer can discover, inherit, and build upon them. The teenager who pulls a combat system from one game into her own project does not need the original studio's source code or permission.

## The Mental Model

1. **Records** — The atoms: generic containers (entities) with IDs and typed Fields.
2. **Fields** — The data: typed values attached to Records, from ultra-granular primitives (e.g., `runs:position_x`) to bundled composites (e.g., `runs:transform`). The type system is open — Processors define whatever input/output shapes their logic requires. The [RUNS Library](https://github.com/enduring-game-standard/runs-library) recommends common shapes for interoperability, but these are a shared vocabulary, not a closed universe.
3. **Processors** — The logic: pure, stateless transformations that read Fields and write Fields. A Processor has no side effects and no hidden state — meaning same inputs always produce the same outputs, with no externally visible mutation between invocations. Internal computation (local variables, scratch buffers, intermediate values) is unrestricted; the contract is at the interface boundary, not inside the implementation. A Processor can be as granular as a vector addition or as bundled as a full character controller.
4. **Networks** — The graph: explicit wiring of Processors into data-flow chains. Wiring may include **guarded Arcs** — transitions conditioned on Field values (e.g., dispatching to different Processors based on an entity's current state). This mirrors MAPS notation, where Arcs carry guard expressions. When two Processors share a data dependency on the same Field, the Network's wiring determines execution order; a linear chain is a valid and common topology. Networks can bundle into higher-scale meta-Processors for multi-level composition.
5. **Runtimes** — The compiler and executor: translates RUNS source (Networks, Records, Processors) into platform-specific builds, manages scheduling, and handles input/output. A minimal interpreter prioritizes portability; a performance-focused implementation compiles and fuses graphs for native speed. Both consume the same RUNS source.
6. **Runtime Interface** — The declared boundary between a Network's game logic and its platform-dependent endpoints. A Network declares the Fields it requires from the runtime (input: timing, player commands) and the Fields it produces for the runtime (output: spatial state, visual state, audio triggers). Swapping one runtime for another leaves the game Network — and everything inside it — unchanged. This is not a new primitive; it is a naming convention on existing components that formalizes portability.

Example loop:

```
[Input Events] → (Input Processor) → [Intent Fields] → (Movement Processor) → [Transform Fields] → (Render Processor) → Screen
```

Swap a naive Movement Processor for a full physics bundle? The chain remains intact as long as shared Fields match.

## A Concrete Example

A Processor is a plain-text definition specifying inputs, outputs, and a pure transformation. The `.runs-prim` format from the [RUNS Library](https://github.com/enduring-game-standard/runs-library) demonstrates how granular these components can be:

```text
processor integrate_velocity
inputs:
  position: vec3
  velocity: vec3
  delta_time: float
outputs:
  position: vec3

position.x = position.x + velocity.x * delta_time
position.y = position.y + velocity.y * delta_time
position.z = position.z + velocity.z * delta_time
```

This Processor does one thing: Euler integration. It reads three Fields, writes one. Anyone can read it, reimplement it, or replace it. Published as a Nostr event, it becomes part of the commons — discoverable by any developer, composable with any Network that uses the same Fields.

Processors wire into bundles. Bundles wire into systems. A character controller bundles movement, grounding, and collision. A game bundles controllers, rendering, and input. Every layer remains a uniform Processor, composable and inspectable from the top-level system down to the individual vector addition.

### Guarded Dispatch

Networks can express data-driven routing through guarded Arcs. This is the RUNS implementation of MAPS guard expressions — a transition fires only when its condition evaluates to true over current Field values.

A state machine (e.g., an enemy AI cycling through look, chase, and attack behaviors) is not a single Processor that internally dispatches. It is a Network of guarded Arcs, each routing to a named Processor:

```yaml
network enemy_behavior
inputs:
  mobj: doom:mobj
outputs:
  mobj: doom:mobj

arcs:
  - guard: mobj.state_action == "look"
    processor: doom:ai_look
  - guard: mobj.state_action == "chase"
    processor: doom:ai_chase
  - guard: mobj.state_action == "attack"
    processor: doom:ai_attack
```

Each branch is a named, inspectable Processor with declared inputs and outputs. The guard expressions and the Processors they route to are all plain-text and visible in the Network graph. Large dispatch tables (hundreds of states) are organized through bundling — sub-Networks grouped by behavior phase, bundled into a meta-Processor for the entity type, composed into the top-level game Network. The decomposition is spaghetti by nature; the bundling hierarchy makes it readable.

## Architecture: Protocol, Library, Ecosystem

RUNS separates what must be universal from what should be shared from what can vary freely.

### Protocol — The Kernel

The mandatory rules: Records, Fields, Processors, Networks, and the `runs:` namespace reservation. A runtime claiming RUNS compliance implements these exactly. The protocol changes rarely and by RFC only.

### Standard Library — The Shared Palette

Curated in [runs-library](https://github.com/enduring-game-standard/runs-library): optional but recommended schemas (e.g., `runs:transform`, `runs:velocity`, `runs:delta_time`) and primitive Processors. Targeting these shapes unlocks instant interoperability — a movement Processor from one bundle drives transforms in another without adapters.

### Ecosystem — The Commons

Community bundles targeting protocol shapes, published as plain-text Nostr events. Anyone can publish a Processor, a Network, or a full game as events that any relay can replicate. No gatekeepers. No app stores. No platform fees. Each contribution becomes part of a permanent, inheritable commons that grows the way open-source package ecosystems grow: through usage, adoption, and variation.

## Connection to Notation and Craft

RUNS is the implementation complement to [MAPS Notation](https://github.com/enduring-game-standard/maps-notation). The relationship is design-to-implementation: MAPS describes a game's rules as studyable notation; RUNS implements those rules as composable source. States in notation are implemented as Records. Verbs are implemented as Processors. Arcs — including their guard expressions — are implemented as Network wiring with guarded transitions. A designer who writes a combat system in MAPS notation is writing the blueprint from which a developer (or tool) builds the corresponding RUNS source. Both artifacts persist independently on the commons — the notation for study, the source for compilation and play.

## What RUNS Deliberately Excludes

RUNS stays restrained and neutral:

- No default renderer, physics, or input model.
- No scripting language — Networks are the composition layer.
- No object hierarchy — pure data composition.
- No network transport — RUNS defines local state transformation. Transport is a separate concern, handled by implementations and coordinated through WOCS.
- No dependency resolution in the hot path — handled by tooling (dynamic resolution for development, static flattening at compile time). The gameplay tick loop is self-contained once compiled.

## Lifecycle: Discover → Build → Run

RUNS components are distributed through Nostr as plain-text events. This is a distribution mechanism, not a runtime dependency.

- **Discovery**: Developers query Nostr relays to find Processors, Networks, Fields, and AEMS Entities/Manifestations. Dependencies are declared in manifests.
- **Build**: A build tool resolves dependencies (from relays or local cache), flattens the Network graph, and compiles the result into a self-contained build. AEMS Manifestations become compiled lookup tables. Processor definitions become compiled functions. Network wiring becomes compiled call chains.
- **Runtime**: The compiled build executes. The gameplay tick loop has no dependency on Nostr, network connectivity, or external services. AEMS Asset and State events may be read/written at lifecycle boundaries (load, save) but not per-frame.

Nostr is the commons — the package registry where source lives. The compiled build is the deployment. The same relationship holds between any package registry and the programs built from its contents.

## Integration with EGS

| Component | Role                           | RUNS Relationship                                              |
|-----------|--------------------------------|----------------------------------------------------------------|
| AEMS      | The things                     | Entity/Manifestation data compiled into type definitions at build time; Asset/State events read/written at lifecycle boundaries (load, save) |
| MAPS      | The rules (notation)           | Design-time blueprint; RUNS source implements MAPS Scores       |
| WOCS      | The coordination               | Coordinates all ecosystem infrastructure — hosting, audits, development, tournaments |

## Comparison

| Feature       | Unity/Unreal      | Bevy/Flecs                   | RUNS                                     |
|---------------|-------------------|------------------------------|------------------------------------------|
| Architecture  | Monolithic        | ECS (code-first)             | Data-first composable substrate          |
| Interop       | Closed            | Runtime-coupled              | Universal plain-text Nostr-native shapes |
| Variation     | Restricted API    | Plugin system                | Open source — swap Processors, compile variants |
| Permanence    | Engine-dependent  | Binary-dependent             | Plain-text portability across generations|
| Distribution  | Stores/platforms  | Git/crates                   | Permissionless (Nostr relays)            |

## Namespace Conventions

Field and Processor keys are namespaced strings (e.g., `prefix:name`) to prevent collisions in a decentralized ecosystem.

- The `runs:` prefix is **reserved** for the RUNS Protocol and Standard Library. No exceptions.
- Runtimes claiming compliance **must** implement `runs:` schemas with exact keys, types, and semantics.
- Third-party bundles use short, human-readable umbrella prefixes (e.g., `pong:`, `hades:`, `spacewar:`).
- Publish bundles as plain-text Nostr events with a manifest anchoring the umbrella prefix, version, and dependencies.

See the [RUNS Library](https://github.com/enduring-game-standard/runs-library) for full namespace conventions and manifest format.

## Summary

RUNS decomposes game logic into composable, plain-text source that lives in an open commons. Records hold state. Processors transform it. Networks wire them together. Nostr makes them discoverable and inheritable without gatekeepers. Combined with AEMS for persistent entities, MAPS for design notation, and WOCS for coordination, RUNS provides the durable substrate on which cumulative craft can compound — one Processor, one Network, one generation at a time.

Implement a Processor. Wire a Network. Build something that endures.

**MIT License** — Open for implementation, extension, critique.