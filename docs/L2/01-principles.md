# L2 — Architectural Principles

These principles shape every container in this folder. The container list is a *consequence* of applying them, not an arbitrary partition.

## Data principles

| Principle | Implication |
|---|---|
| Atoms (claims) are canonical SoR | Persisted in a diffable form. Flat with relations expressed as RelationAtoms, never nested. |
| Prose projections are derived | Exposed through the projection read facet each contentful container implements over its own content. Derived from atoms; never authoritative. |
| Tree partitioning is structural | Same-claim atoms collapse within a tree; cross-tree parallel claims coexist via `kind=equivalence`. Trees are the unit of bounded discourse. |
| Classification + predicate schemas drive validation | Structural contracts on classifications and predicates make conflict pipeline checks tractable. |
| Cascade rides three channels | Predicate-kind cascade (per kind rules), scope-premise cascade (via `Scope.constraint_atoms`), and derivation invalidation. The same machinery powers both lifecycle transitions and Blast Radius analyses. |
| Entailment is first-class | Derived atoms (T1+T2+T3) are part of the snapshot. Hidden by default in the projection facet; transparently consumed by Conflict and Blast Radius, and available to external agents through the read surfaces. |

## Architecture principles (container partitioning)

| Principle | Implication |
|---|---|
| **Capabilities as containers** | Each L2 container is a single concern that owns its own state and exposes its own R/W APIs. |
| **Federated state** | Each container owns its own state. Consumers reach state by going through the owning container's API. (CQRS-shaped.) |
| **State is private to its owner; access is API-only** | The principle applies at every level — containers, and components inside a container. State (atom store, indexes, caches, reports, sessions) can be read or mutated only through the operations the owner exposes. No external entity reaches around the API to touch internal storage. This is what makes "the only component that mutates X" claims load-bearing — they're enforced by the principle, not restated as exceptions per consumer. |
| **A capability earns its own L2 slot if any one of these holds:** | |
| (a) it owns substantial derived state | E.g., Blast Radius (reports, traversal indexes), Hierarchical Navigation (multi-axis tree indexes). |
| (b) it does significant derivative/composition work | E.g., Blast Radius traverses the cascade channels to compose an impact report from many atoms. |
| (c) it is optional / pluggable | E.g., a given deployment can ship without Blast Radius. |
| **Otherwise, it's a primitive** | If a "capability" has no state and no significant derivative work, it's a primitive of whoever owns the underlying data. E.g., alias resolution is a primitive on ClassificationAtoms and PredicateAtoms and lives inside Atoms; relation traversal is a primitive on RelationAtoms and lives inside Atoms. |
| **Core capabilities must not require optional ones** | If a non-optional container depends on an optional one, something is misclassified: either promote the optional, fold the dependency into a primitive, or break the dependency. |
| **Optional-on-optional is fine *when degradation is graceful*** | A capability may consume other optional capabilities if it can degrade to a lower-fidelity path when they're absent. The minimum-viable input must be specified per capability. |
| **Projection is a cross-cutting read facet, not a container** | A contentful container exposes two read surfaces: its structured read API (graph queries) and a read-only projection facet over its own content (canonical prose + a navigable index). Required of Atoms; optional for Ingestion, Blast Radius, and Hierarchical Navigation. See [`04-projection.md`](./04-projection.md). |
| **Each container exposes its own diff** | "What changed" is owned by the producer of the change. The agent (or any UI client) aggregates across container diff APIs to present a unified view. |
| **Containers expose ordered delta streams** | Every container holding state emits an ordered, typed delta stream via `changes_since(ref)`. Downstream consumers maintain their derived state by consuming the stream in order, applying deltas incrementally — not by periodic full rebuilds. Contract: `snapshot(ref_X) + deltas(X → Y) ≡ snapshot(ref_Y)`. Full replay from scratch always works, so a consumer that's lost state can catch up. |
| **Global L2/L3 documents the superset of capabilities; implementations realize subsets** | Specific deployable shapes (local-MCP, SaaS multi-tenant, etc.) live under `docs/implementations/<impl>/` and reference the same interfaces. |
| **Interfaces are first-class artifacts** | Each container's interface contract lives in `interfaces/<container>.md`, versionable on its own. |
| **Non-logical metadata is named as such** | `annotations`, `attributes`, `confidence`, and `Scope.time_scope` carry record-level metadata explicitly excluded from logical reasoning (Conflict, Blast Radius, entailment). `annotations` is the OWL annotation-property analog (free-form, never validated); `attributes` holds schema-validated typed values, likewise excluded from reasoning. |
