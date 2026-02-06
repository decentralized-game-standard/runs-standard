# RUNS: Record Update Network System

🏠 **[DGS Overview](https://github.com/decentralized-game-standard)**  
· 📦 **[AEMS](https://github.com/decentralized-game-standard/aems-standard)**  
· ⚡ **[WOCS](https://github.com/decentralized-game-standard/wocs-standard)**  
· 🔤 **[Ludic](https://github.com/decentralized-game-standard/ludic-notation-standard)**  
· ❓ **[FAQ](https://github.com/decentralized-game-standard/.github/blob/main/profile/FAQ.md)**

> **Status**: Draft / RFC  
> **Version**: 0.1.0

## The Neutral Substrate for Enduring Games

**RUNS** (Record Update Network System) is the execution model that enables games to outlive engines, companies, and hardware cycles.

Inspired by Unix pipes and data-oriented design, RUNS treats game engines as composable data-flow graphs: uniform **Records** hold state, stateless **Processors** transform data, and explicit **Networks** wire everything together. Components evolve independently—mods become native extensions, obsolescence in one layer never cascades, and diverse runtimes (minimal web builds to high-performance native) share the same ecosystem like Linux modules.

RUNS is not a full engine or rule language. It is the minimal "kernel" layer: a neutral substrate for transforming data, with game-specific logic implemented in swappable Processors and coordinated across the broader Decentralized Game Standard (DGS).

For extreme longevity and permissionless distribution, distributable artifacts—Networks, Records, Processors, and ecosystem packages—are plain-text Nostr events by convention. This ensures seamless, tamper-proof replication across relays without binaries, central hosting, or gatekeepers. Provenance chains through note IDs enable self-describing data that survives centuries.

## Why RUNS?

Current engines are monolithic and fragile:

- Inheritance of one company's physics, rendering, and fate.
- Mods limited to restricted scripting.
- States tied to dying executables.

RUNS unbundles everything:

- **Universal Modding** — Mods are new Processors or Network extensions with full privileges.
- **Extreme Portability** — Logic runs on any compliant runtime (web, desktop, embedded).
- **Cultural Permanence** — Serializable Records and plain-text Networks allow states to migrate across decades—resume a game long after the original runtime is gone.
- **True Interoperability** — Shared data shapes let Processors from different developers compose freely.
- **Permissionless Ecosystem** — Plain-text Nostr events enable decentralized discovery, forking, and coordination.
- **Rapid Remix Prototyping** — High-level bundles (e.g., `hades:combat` + `zelda-2d:map`) resolve dynamically for instant playable sketches, then flatten to optimized binaries for polished release.

## The Mental Model

1. **Records** — The atoms: generic containers (entities) with IDs and Fields of state.
2. **Fields** — The data: typed values on Records, from ultra-granular primitives (e.g., `position_x`) to bundled composites.
3. **Processors** — The logic: pure, stateless transformations reading/writing Fields. Granular like syscalls or hierarchically bundled.
4. **Networks** — The graph: explicit wiring of Processors into data-flow chains. Networks can bundle into higher-scale "meta-Processors" for multi-level composition.
5. **Runtimes** — The executor: loads Networks, manages scheduling, handles input/output. Two-tier execution: dynamic resolution for prototyping/remix, compile-time flattening for per-platform performance binaries.

Example simple loop:

```
[Input Events] → (Input Processor) → [Intent Fields] → (Movement Processor) → [Transform Fields] → (Render Processor) → Screen
```

Swap a naive Movement Processor for a full physics bundle? The chain remains intact as long as shared Fields match.

## What RUNS Enables

- **Multi-Scale Composition** — Primitives wire into mid-level bundles (e.g., velocity integration from raw axes), which bundle into systems (e.g., character controller), all remaining uniform Processors with provenance chains to the atoms.
- **Diverse Runtimes** — Minimal interpreters prioritize portability; pro implementations fuse graphs for bleeding-edge performance—all from the same Networks.
- **Commons of Remixable Pigments** — Ultra-granular primitives bundle into reusable composites (like master painters mixing paints), distributed as Nostr events. Communities remix ad nauseam—prototype novel games in hours by combining high-level bundles, then build to optimized executables.
- **DGS Integration** — WOCS coordinates Processor services and curation; AEMS provides entity definitions via parallel Nostr ecosystem; Nostr distributes plain-text packages permissionlessly.

## What RUNS Deliberately Excludes

RUNS stays restrained and neutral:

- No default renderer, physics, or input model.
- No scripting language—Networks are the composition script.
- No object hierarchy—pure data composition.
- No network transport—focus on local-first state diffs.
- No runtime dependency resolution—handled by tooling (dynamic for dev, static flattening for builds).

## Namespace Conventions

Field and Processor keys are namespaced strings (e.g., `prefix:name`). This prevents collisions in a fully compositional, decentralized ecosystem.

### Non-Negotiable Rules (Protocol Level)

- The `runs:` prefix is **reserved exclusively and forever** for the RUNS Protocol and Standard Library schemas.
- No exceptions, no grandfathering, no custom usage allowed.
- Runtimes claiming RUNS compliance **must** implement standard `runs:` schemas with exact keys, field names, types, and semantics.
- Deviation means the runtime or package opts out of the shared ecosystem entirely.
- Core `runs:` schemas are pinned to this specification and the Standard Library repo. Additions require RFC and overwhelming justification for universality.

### Recommended Conventions (Ecosystem Bundles)

Third-party bundles **must not** use `runs:`. Instead:

- Use short, human-readable **umbrella prefixes** as reusable brands (e.g., `pong:`, `hades:`, `zelda-oot:`).
- Publish as **plain-text Nostr events** with a manifest anchoring the umbrella:
  ```json
  {
    "umbrella": "pong:",                  // Reusable brand prefix (required)
    "version": "1.4.3",                    // Semantic version (required)
    "note_id": "note1xyz...abc",           // This event's ID (content-addressed anchor, required)
    "previous_note_id": "note1def...456",  // Optional lineage
    "author_npub": "npub1...",             // Optional provenance
    "components": ["health", "velocity"],  // Exported fields/processors (required)
    "dependencies": { /* umbrella@version or note IDs */ }
  }
  ```
- Data uses clean keys: `{ "pong:health": 100 }`.
- Tooling resolves prefixes via manifests—detects conflicts, enables aliasing.
- Optional: Append `@version` to keys for explicit pinning in mixed compositions.
- Bundles chain provenance via dependencies/note IDs—self-describing from high-level systems to granular primitives.

These conventions preserve readability while ensuring cryptographic safety and centuries-scale endurance.

## Architecture: Purity in the Protocol, Flexibility in the Library

### Protocol – The Eternal Kernel

Unbreakable rules ensuring interoperability and decentralization.

### Standard Library – The Shared Palette

Curated in [runs-standard-library](https://github.com/decentralized-game-standard/runs-standard-library): optional but encouraged schemas (e.g., `runs:transform`, `runs:time`) for instant interoperability.

### Ecosystem – Where Flexibility Flourishes

Community bundles targeting Protocol shapes, distributed as Nostr events.

## Integration with DGS Standards

| Standard | Role                          | RUNS Relationship                          |
|----------|-------------------------------|--------------------------------------------|
| AEMS     | Persistent artifacts          | Referenced for entity data                 |
| Ludic    | Rule descriptions             | Implemented via Processors                 |
| WOCS     | Coordination/services/curation| Funds Processor development, relay hosting, and bundle ranking |

## Comparison

| Feature       | Unity/Unreal | Bevy/Flecs     | RUNS                                      |
|---------------|--------------|----------------|-------------------------------------------|
| Architecture  | Monolithic   | ECS (code-first)| Data-first neutral substrate              |
| Interop       | Closed       | Language-locked| Universal plain-text Nostr-native shapes  |
| Modding       | Restricted API| Compile-time plugins| Runtime Network injection            |
| Permanence    | Engine-dependent| Binary-dependent| Data/Network portability across centuries|
| Distribution  | Stores/platforms| Git/crates  | Permissionless (Nostr relays)             |

## Summary

RUNS is the durable socket for decentralized games: explicit, swappable, plain-text native, provenance-chained. Combined with DGS layers, it elevates game development to nuance—engineers engineer the substrate, designers remix enduring pigments into novel experiences that evolve indefinitely without capture.

Implement a Processor. Wire a Network. Build something that endures.

**MIT License** — Open for implementation, extension, critique.