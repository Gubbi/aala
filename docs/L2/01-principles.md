# L2 — Architectural Principles

These principles shape every container in this folder. The container list is a *consequence* of applying them, not an arbitrary partition.

## Data principles

| Principle | Implication |
|---|---|
| Atoms (claims) are canonical SoR | YAML in git repo. Flat with references, never nested. |
| Prose projections are derived | Human review surface only. Regenerated from atoms; never authoritative. |
| Premise tags + status field on atoms | Make blast-radius analysis on sweeping decisions tractable. |

## Architecture principles (container partitioning)

| Principle | Implication |
|---|---|
| **Capabilities as containers** | Each L2 container is a single concern that owns its own state and exposes its own R/W APIs. |
| **Federated state** | Each container owns its own state. Consumers reach state by going through the owning container's API. (CQRS-shaped.) |
| **State is private to its owner; access is API-only** | The principle applies at every level — containers, and components inside a container. State (atom store, indexes, caches, reports, sessions) can be read or mutated only through the operations the owner exposes. No external entity reaches around the API to touch internal storage. This is what makes "the only component that mutates X" claims load-bearing — they're enforced by the principle, not restated as exceptions per consumer. |
| **A capability earns its own L2 slot if any one of these holds:** | |
| (a) it owns substantial derived state | E.g., Blast Radius (reports, traversal indexes), Hierarchical Navigation (multi-axis tree indexes). |
| (b) it does significant derivative/composition work | E.g., Synthesis composes many APIs into one response. |
| (c) it is optional / pluggable | E.g., a given deployment can ship without Blast Radius. |
| **Otherwise, it's a primitive** | If a "capability" has no state and no significant derivative work, it's a primitive of whoever owns the underlying data. E.g., alias resolution is a primitive on `DefinitionAtom` and lives inside Atoms; entity-reference traversal is a primitive on atom fields and lives inside Atoms. |
| **Core capabilities must not require optional ones** | If a non-optional container depends on an optional one, something is misclassified: either promote the optional, fold the dependency into a primitive, or break the dependency. |
| **Optional-on-optional is fine *when degradation is graceful*** | A capability may consume other optional capabilities if it can degrade to a lower-fidelity path when they're absent. The minimum-viable input must be specified per capability. |
| **Each container exposes its own diff** | "What changed" is owned by the producer of the change. The agent (or any UI client) aggregates across container diff APIs to present a unified view. |
| **Containers expose ordered delta streams** | Every container holding state emits an ordered, typed delta stream via `changes_since(ref)`. Downstream consumers maintain their derived state by consuming the stream in order, applying deltas incrementally — not by periodic full rebuilds. Contract: `snapshot(ref_X) + deltas(X → Y) ≡ snapshot(ref_Y)`. Full replay from scratch always works, so a consumer that's lost state can catch up. |
| **Global L2/L3 documents the superset of capabilities; implementations realize subsets** | Specific deployable shapes (local-MCP, SaaS multi-tenant, etc.) live under `docs/implementations/<impl>/` and reference the same interfaces. |
| **Interfaces are first-class artifacts** | Each container's interface contract lives in `docs/interfaces/<container>.md`, versionable on its own. |
