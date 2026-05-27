# Hierarchical Navigation — Interface Contract

**Version:** `aala-v1`
**Optionality:** optional
**Container:** [`L2/06-hierarchical-nav.md`](../L2/06-hierarchical-nav.md) · [`L3/06-hierarchical-nav.md`](../L3/06-hierarchical-nav.md)

Hierarchical Navigation builds and serves multi-axis labeled tree indexes over atoms. Each axis is a different organizing principle (by-domain, by-team, by-lifecycle, by-tag); they share an interface shape.

## Types

```typescript
type AxisId = string  // e.g., "by-domain", "by-team", "by-lifecycle", "by-tag"
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
  atoms:          AtomId[]           // atoms living at this node
}
```

## Methods

### `axes`

```typescript
function axes(): Promise<AxisMeta[]>
```

Which axes this deployment has built. Consumers (especially Synthesis) use this for graceful degradation.

**Errors:** `CommonError`.

---

### `navigate`

```typescript
function navigate(axis_id: AxisId, path: NodePath): Promise<TreeNode | null>
```

Fetch one node of one tree. Returns `null` if the path doesn't resolve.

**Errors:** `CommonError | { kind: "UnknownAxis"; axis_id: AxisId }`.

---

### `locate`

```typescript
function locate(axis_id: AxisId, atom_id: AtomId): Promise<NodePath | null>
```

Given an atom, find where it sits in this axis's tree. Returns `null` if the atom is not placed in this axis.

**Errors:** `CommonError | { kind: "UnknownAxis"; axis_id: AxisId }`.

---

### `rebuild`

```typescript
function rebuild(axis_id?: AxisId, scope?: NodePath): Promise<RebuildResult>

interface RebuildResult {
  axis_id:           AxisId
  nodes_added:       number
  nodes_changed:     number
  nodes_removed:     number
  labels_synthesized:number
  integrity_report:  IntegrityReport
}

interface IntegrityReport {
  orphans:        AtomId[]       // atoms not placed in any node
  dead_refs:      AtomId[]       // tree refs to atoms that no longer exist
}
```

Regenerate the tree (or scoped subtree) from atoms in the current snapshot. With no axis: rebuilds all axes. With an axis but no scope: rebuilds the whole axis. With axis + scope: rebuilds only the affected subtree.

Typically triggered by Orchestration on snapshot change. Steady-state operation uses incremental deltas via Delta Consumer; `rebuild` is for full recomputation.

**Errors:** `CommonError | { kind: "UnknownAxis"; axis_id: AxisId }`.

**Idempotency:** idempotent given the same snapshot — repeated calls produce the same tree state.

**Concurrency:** serialized per-axis. Different axes may rebuild concurrently.

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

- **`aala-v1`** — initial contract.
- Patch-compatible: new event kinds, new `AxisMeta` fields, new fields in `TreeNode`.

## Notes for implementers

- The axis set is deployment configuration (which axes are wired) and is not part of the contract.
- Label synthesis goes through LLM Gateway with `aala.label_synthesis`. Configurations without LLM Gateway may use rule-based labels — labels still flow through Tree Store and Change Log.
- Tree persistence is implementation choice: alongside atoms in the snapshot, or container-internal cache.
