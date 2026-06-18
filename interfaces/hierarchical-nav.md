# Hierarchical Navigation — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** optional

Hierarchical Navigation builds and serves multi-axis labeled tree indexes over atoms. Each axis is a different organizing principle (by-classification, by-team, by-lifecycle, by-tag, by-tree); they share an interface shape.

## Types

```typescript
type AxisId = string  // e.g., "by-classification", "by-team", "by-lifecycle", "by-tag", "by-tree"
type NodePath = string[]  // path segments from the axis root

interface AxisMeta {
  axis_id:        AxisId
  display_name:   string
  description:    string
  organizing_by:  string   // field / metadata the axis derives shape from
}

interface TreeNode {
  axis_id:        AxisId
  path:           NodePath
  label:          string             // synthesized human-readable label
  description?:   string             // synthesized longer description
  children:       NodePath[]
  atom_count:     number             // atoms living at this node; the membership itself is read via `node_atoms` (paged)
}
```

`TreeNode` is bounded: it carries the node's metadata, child paths, and a membership **count** — never the membership list. A hot node (a by-lifecycle bucket, a large tree's by-tree node) can hold most of the corpus; its members are enumerated through the paged [`node_atoms`](#node_atoms).

The `by-classification` axis is built directly from ClassificationAtom hierarchies — it reflects the `is_a` chain structure as the navigation hierarchy. The `by-tree` axis groups atoms by `Scope.tree`. Other axes (by-team, by-lifecycle, by-tag) derive shape from atom attributes or RelationAtoms.

## Methods

### `axes`

```typescript
function axes(): Promise<AxisMeta[]>
```

Which axes this deployment has built. Consumers (external agents, Blast Radius) use this for graceful degradation.

**Errors:** `navigating` — universal end-states only.

**Access:** `query` ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `navigate`

```typescript
function navigate(axis_id: AxisId, path: NodePath): Promise<TreeNode | null>
```

Fetch one node of one tree. Returns `null` if the path doesn't resolve.

**Errors:** `navigating` — `unknown_axis` (the named axis is not registered).

**Access:** `query`.

---

### `node_atoms`

```typescript
function node_atoms(
  axis_id:  AxisId,
  path:     NodePath,
  page?:    PageRequest
): Promise<Page<AtomId>>
```

Enumerate the atoms living at one node — the paged membership read behind `TreeNode.atom_count` ([00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads)). A path that does not resolve yields an empty enumeration (the members of an absent node form the empty set); callers that need to distinguish an absent node from an empty one check existence with `navigate`. Hydrate the returned ids with [`Atoms.get_by_ids`](./atoms.md#get_by_ids).

**Errors:** `navigating` — `unknown_axis` (the named axis is not registered), `invalid_query` (invalid paging cursor).

**Access:** `query`.

---

### `locate`

```typescript
function locate(axis_id: AxisId, atom_id: AtomId): Promise<NodePath | null>
```

Given an atom, find where it sits in this axis's tree. Returns `null` if the atom is not placed in this axis.

**Errors:** `navigating` — `unknown_axis` (the named axis is not registered).

**Access:** `query`.

---

### `rebuild`

```typescript
function rebuild(axis_id?: AxisId, scope?: NodePath): Promise<RebuildResult>

interface RebuildResult {
  axis_id:             AxisId
  nodes_added:         number
  nodes_changed:       number
  nodes_removed:       number
  labels_synthesized:  number
  integrity_report:    IntegrityReport
}

interface IntegrityReport {
  orphans:        AtomId[]       // atoms not placed in any node
  dead_refs:      AtomId[]       // tree refs to atoms that no longer exist
}
```

Regenerate the tree (or scoped subtree) from atoms in the current snapshot. With no axis: rebuilds all axes. With an axis but no scope: rebuilds the whole axis. With axis + scope: rebuilds only the affected subtree.

Typically triggered by Orchestration on snapshot change. Steady-state operation uses incremental deltas via Delta Consumer; `rebuild` is for full recomputation.

**Errors:** `navigating` — `unknown_axis` (the named axis is not registered).

**Access:** `maintain` — full recomputation is a maintenance operation; internally-triggered rebuilds run in `system:`-correlated sessions ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

**Idempotency:** naturally idempotent (pure recomputation) — the rebuilt state is a pure function of the bound snapshot; repeated calls converge to the same tree state. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per-axis. Different axes MAY rebuild concurrently.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type HierarchicalNavEvent =
  | { kind: "TreeNodeAdded";    payload: { axis_id: AxisId; path: NodePath; node: TreeNode } }
  | { kind: "TreeNodeChanged";  payload: { axis_id: AxisId; path: NodePath; before: TreeNode; after: TreeNode } }
  | { kind: "TreeNodeRemoved";  payload: { axis_id: AxisId; path: NodePath; reason: string } }
  | { kind: "LabelUpdated";     payload: { axis_id: AxisId; path: NodePath; label: string; description?: string } }
```

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible: new event kinds, new `AxisMeta` fields, new fields in `TreeNode`.

## Notes for implementers

- The axis set is deployment configuration (which axes are wired) and is not part of the contract.
- Label synthesis goes through LLM Gateway with `aala.label_synthesis`. Configurations without LLM Gateway MAY use rule-based labels — labels still flow through the tree store and the Delta Stream.
- Tree persistence is implementation choice: alongside atoms in the snapshot, or container-internal cache.
- The `by-classification` axis MUST reflect the ClassificationAtom `is_a` hierarchy. Other axes derive from attribute values, RelationAtoms, or external metadata per the axis configuration.
- The Hierarchical Navigation "tree" (an axis tree) is distinct from the `TreeId` partition on `Scope.tree`. The `by-tree` axis is one specific axis that groups atoms by `Scope.tree` value.
