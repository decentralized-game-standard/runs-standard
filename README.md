# RUNS: Record Update Network System

🏠 **[EGS Overview](https://github.com/enduring-game-standard)**  
· 📦 **[AEMS](https://github.com/enduring-game-standard/aems-schema)**  
· ⚡ **[WOCS](https://github.com/enduring-game-standard/wocs-protocol)**  
· 🎼 **[MAPS](https://github.com/enduring-game-standard/maps-notation)**  
· ❓ **[FAQ](https://github.com/enduring-game-standard/.github/blob/main/profile/FAQ.md)**

> **Status**: Draft / RFC  
> **Version**: 0.1.0

## A Composable Substrate for Enduring Games

**RUNS** (Record Update Network System) is a composable, plain-text architecture for game execution. Stateless Processors transform Records through explicit Networks, giving games a neutral substrate designed to outlive any single engine, company, or hardware cycle.

RUNS treats game engines the way Unix treats operating systems: as pipelines of small, focused components wired together. Records hold state. Processors transform it. Networks define the wiring. Each piece evolves independently. Mods are not restricted scripts — they are Processors with full privileges. Obsolescence in one layer never cascades to another. Diverse runtimes, from minimal web builds to high-performance native, share the same ecosystem.

RUNS components — Networks, Records, Processors, and ecosystem packages — are plain-text Nostr events by convention. This is not a deployment detail. Nostr is the commons: the mechanism by which composable game components become discoverable, inheritable, and remixable across generations. Provenance chains through note IDs ensure self-describing data that any future community can find, study, and build upon without permission.

RUNS is not a full engine or rule language. It is the minimal kernel layer: a neutral substrate for transforming data, with game-specific logic implemented in swappable Processors and coordinated across the broader Enduring Game Standard (EGS).

## Why RUNS?

Current engines are monolithic and fragile:

- Inheritance of one company's physics, rendering, and fate.
- Mods limited to restricted scripting.
- Game logic locked inside opaque binaries that no one outside the studio can read, maintain, or learn from.

RUNS unbundles everything:

- **Universal Modding** — Mods are new Processors or Network extensions with full privileges.
- **Portability** — Logic runs on any compliant runtime (web, desktop, embedded).
- **Interoperability** — Shared data shapes let Processors from different developers compose freely.
- **Open Commons** — Plain-text Nostr events place components in an open commons where any developer can discover, inherit, and build upon them. The teenager who pulls a combat system from one game into her own project does not need the original studio's source code or permission.

## The Mental Model

1. **Records** — The atoms: generic containers (entities) with IDs and typed Fields.
2. **Fields** — The data: typed values attached to Records, from ultra-granular primitives (e.g., `runs:position_x`) to bundled composites (e.g., `runs:transform`). Fields define the shared vocabulary that makes composition possible.
3. **Processors** — The logic: pure, stateless transformations that read Fields and write Fields. A Processor has no side effects and no hidden state. It can be as granular as a vector addition or as bundled as a full character controller.
4. **Networks** — The graph: explicit wiring of Processors into data-flow chains. Networks can bundle into higher-scale meta-Processors for multi-level composition.
5. **Runtimes** — The executor: loads Networks, manages scheduling, handles input/output. A minimal interpreter prioritizes portability; a performance-focused implementation fuses graphs for native speed. Both run the same Networks.

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

## Architecture: Protocol, Library, Ecosystem

RUNS separates what must be universal from what should be shared from what can vary freely.

### Protocol — The Kernel

The mandatory rules: Records, Fields, Processors, Networks, and the `runs:` namespace reservation. A runtime claiming RUNS compliance implements these exactly. The protocol changes rarely and by RFC only.

### Standard Library — The Shared Palette

Curated in [runs-library](https://github.com/enduring-game-standard/runs-library): optional but recommended schemas (e.g., `runs:transform`, `runs:velocity`, `runs:delta_time`) and primitive Processors. Targeting these shapes unlocks instant interoperability — a movement Processor from one bundle drives transforms in another without adapters.

### Ecosystem — The Commons

Community bundles targeting protocol shapes, published as plain-text Nostr events. Anyone can publish a Processor, a Network, or a full game as events that any relay can replicate. No gatekeepers. No app stores. No platform fees. Each contribution becomes part of a permanent, inheritable commons that grows the way open-source package ecosystems grow: through usage, adoption, and variation.

## Connection to Notation and Craft

RUNS is the runtime complement to the [MAPS Notation](https://github.com/enduring-game-standard/maps-notation). The connection is concrete: States in notation become Records that RUNS processes. Verbs become Processors that transform them. Arcs become Networks that wire them together. A designer who sketches a combat system in notation is writing the skeleton that the composable infrastructure reads. MAPS describes the rules; RUNS executes the game.

## What RUNS Deliberately Excludes

RUNS stays restrained and neutral:

- No default renderer, physics, or input model.
- No scripting language — Networks are the composition layer.
- No object hierarchy — pure data composition.
- No network transport — RUNS defines local state transformation. Transport is a separate concern, handled by implementations and coordinated through WOCS.
- No runtime dependency resolution — handled by tooling (dynamic for development, static flattening for builds).

## Integration with EGS

| Component | Role                           | RUNS Relationship                                              |
|-----------|--------------------------------|----------------------------------------------------------------|
| AEMS      | The things                     | Referenced for entity data                                     |
| MAPS      | The rules                      | Notation targets Records, Processors, Networks                 |
| WOCS      | The coordination               | Coordinates all ecosystem infrastructure — hosting, audits, development, tournaments |

## Comparison

| Feature       | Unity/Unreal      | Bevy/Flecs                   | RUNS                                     |
|---------------|-------------------|------------------------------|------------------------------------------|
| Architecture  | Monolithic        | ECS (code-first)             | Data-first composable substrate          |
| Interop       | Closed            | Runtime-coupled              | Universal plain-text Nostr-native shapes |
| Modding       | Restricted API    | Plugin system                | Runtime Network injection                |
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

RUNS decomposes game execution into composable, plain-text primitives that live in an open commons. Records hold state. Processors transform it. Networks wire them together. Nostr makes them discoverable and inheritable without gatekeepers. Combined with AEMS for persistent entities, MAPS for design notation, and WOCS for coordination, RUNS provides the durable substrate on which cumulative craft can compound — one Processor, one Network, one generation at a time.

Implement a Processor. Wire a Network. Build something that endures.

**MIT License** — Open for implementation, extension, critique.