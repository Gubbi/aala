# Hierarchical Navigation — Multi-Axis Tree Indexes

**Status: optional.** Deployments without this container can still answer queries by walking atoms directly via [Atoms](./03-atoms.md); they pay an O(n) cost for navigation-shaped questions instead of getting an indexed descent.

## Concern

Hierarchical Navigation builds and serves *labeled tree indexes* over the atom set, with multiple axes layered over the same atoms. Each axis is a different organizing principle but they all expose the same shape: descend a labeled tree, get the children and atoms at any node.

The container exists because building those trees is real work — synthesizing semantic labels for nodes, maintaining incremental rebuilds on snapshot changes, supporting LLM-driven descent. That's substantial derived state that justifies its own owner.

In one sentence: **Hierarchical Navigation gives LLMs and humans a labeled-tree way into the atom set, with multiple lenses on the same atoms.**

## Axes

Common axes:

| Axis | Organizes by |
|---|---|
| `by-classification` | The ClassificationAtom `is_a` hierarchy. Atoms cluster under their primary classification's chain. |
| `by-tree` | `Scope.tree` value. Atoms cluster by bounded discourse. |
| `by-domain` | A configured "domain" tag or scope-path segment. |
| `by-team` | A configured "owning team" attribute. |
| `by-lifecycle` | Status state buckets. |
| `by-tag` | Arbitrary tags from atom attributes. |

The `by-classification` axis is special: it derives its hierarchy directly from `is_a` chains and reflects the ClassificationAtom structure. The `by-tree` axis groups by `Scope.tree` — this is distinct from the broader "tree partition" concept (which is the `TreeId` value on every atom); the `by-tree` axis is just one specific Hierarchical-Navigation axis that surfaces the partition.

## What it owns

| Concern | Notes |
|---|---|
| Per-axis tree indexes | One tree per axis. Same atoms, different paths through them. Pinned to the snapshot they were built from. |
| Synthesized labels | LLM-assisted human-readable labels on internal tree nodes (e.g., "Payments domain — handles transaction lifecycle"). Critical for LLM-driven descent. |
| Rebuild pipeline | The machinery that builds / rebuilds a tree from atoms when the snapshot changes. Supports scoped rebuilds (only affected subtrees) for efficiency. |
| Integrity checks | Orphan atoms (atoms not placed in any tree), dead refs (tree nodes pointing at deleted atoms), structural rot detection. |

The fact that multiple axes share one container is deliberate: they share an identical interface shape (`navigate(axis, path)`), differing only in which atom fields / classifications drive the tree shape and label synthesis.

## API surface (conceptual)

**Read:**
- `axes() → [axis_meta]` — which axes this deployment has built.
- `navigate(axis_id, path) → tree_node` — fetch one node of one tree. Returns the node's label, its child node references, and atom IDs that live at this node.
- `locate(axis_id, atom_id) → path` — given an atom, find where it sits in this axis's tree.
- `changes_since(ref)` — what tree nodes changed since the marker. The container's own change stream.

**Write (driven by consumption of the Atoms delta stream):**
- `apply_deltas(stream)` — steady-state path. Consumes events from [Atoms](./03-atoms.md)'s `changes_since(ref)` stream and updates the affected tree nodes incrementally.
- `rebuild(axis_id?, scope?)` — full / scoped rebuild from scratch. Used on initial bootstrap, when an axis's checkpoint is lost or corrupted, or for a forced regeneration.

## Dependencies

- **[Atoms](./03-atoms.md)** — read API; rebuild walks atoms (and their ClassificationAtoms / RelationAtoms) to construct trees.
- **[LLM Gateway](./09-llm-gateway.md)** — label synthesis for human / LLM-readable node names. A configuration without LLM Gateway can use rule-based labels (less fluent, but functional).
- No dependencies on other optional containers.

## Components (preview of L3)

- **Axis Builders** — one per axis. Each knows how to derive its tree shape from atoms (typically by walking is_a chains, scope paths, attribute values, or RelationAtoms) and how to apply individual delta events incrementally.
- **Label Synthesizer** — LLM-assisted node-label generation. Called by every axis builder for newly-created or significantly-changed nodes.
- **Delta Consumer / Rebuild Coordinator** — subscribes to Atoms's `changes_since(ref)` stream; per-axis, tracks the last consumed `ref`; routes events to the appropriate axis builders. Also handles full rebuild from scratch when invoked.
- **Integrity Checker** — runs after applying a delta batch (or after a full rebuild); produces a structural-rot report.
- **Navigation API** — serves `navigate` / `locate` calls against the built trees.
- **Change Log** — maintains the ordered, append-only event log for this container.

Full L3 detail lives in [`docs/L3/06-hierarchical-nav.md`](../L3/06-hierarchical-nav.md).

## Variation points (where implementations differ)

| Variation | Examples |
|---|---|
| Axis set | Minimal (by-classification only) → typical (by-classification + by-tree + by-team) → full (+ by-lifecycle, by-tag, custom). |
| Rebuild mode | Eager on every snapshot change; lazy on first navigation after a change; scheduled (e.g., hourly); on-demand only. |
| Label synthesis | LLM-based with caching (default); rule-based labels from atom metadata; no labels (raw path segments). |
| Integrity check depth | None; basic (orphans + dead refs); deep (consistency between axes, cycle detection). |
| Persistence | Trees stored alongside atoms in the snapshot; or held in container-internal storage (cache only). |

## What Hierarchical Navigation does NOT do

- Alphabetical or alias-based lookup — that's split between [Atoms](./03-atoms.md) (alias resolution primitives via `resolve_subject`) and the [Projection](./04-projection.md) facet (the rendered glossary).
- Relation-graph traversal (RelationAtom edges, predicate-kind filtering) — that's a primitive of [Atoms](./03-atoms.md) (`traverse_relations`), not a tree.
- Answering queries — external agents (and [Blast Radius](./07-blast-radius.md)) navigate this tree when present and fall back to raw Atoms traversal when not; composing answers is the consumer's concern.
- Deciding what an "axis" *means* — axes are configured. Hierarchical Navigation builds and serves; it doesn't invent organizing principles.
