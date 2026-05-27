# Orchestration — Interface Contract

**Version:** `aala-v1`
**Optionality:** non-optional
**Container:** [`L2/05-orchestration.md`](../L2/05-orchestration.md) · [`L3/05-orchestration.md`](../L3/05-orchestration.md)

Orchestration is the outside-world entry point. It owns the snapshot lifecycle and dispatches requests to capability containers.

## Snapshot management

### `current_snapshot`

```typescript
function current_snapshot(): Promise<SnapshotRef>
```

Returns the working snapshot every capability container currently operates against.

**Errors:** `CommonError`.

---

### `fork`

```typescript
function fork(from?: SnapshotId): Promise<SnapshotRef>
```

Create a new working snapshot, optionally forked from a specific source snapshot (default: from canonical).

**Errors:**
```typescript
type ForkError =
  | CommonError
  | { kind: "ParentNotFound"; from: SnapshotId }
```

---

### `switch_to`

```typescript
function switch_to(snapshot_id: SnapshotId): Promise<void>
```

Make the given snapshot the current working one. All subsequent reads/writes against capability containers operate on this snapshot. Triggers downstream consumers (Hierarchical Navigation, Projection, Blast Radius) to re-baseline their `changes_since(ref)` checkpoints.

**Errors:**
```typescript
type SwitchError =
  | CommonError
  | { kind: "SnapshotNotFound"; snapshot_id: SnapshotId }
  | { kind: "SnapshotDiscarded"; snapshot_id: SnapshotId }
```

---

### `publish_as_canonical`

```typescript
function publish_as_canonical(snapshot_id: SnapshotId): Promise<SnapshotRef>
```

Promote a working snapshot to canonical. In deployments where canonicalization happens outside aala (e.g., the user committing via git), this method is a no-op or detection-only; aala observes the external promotion and updates its view. The contract is "the returned `SnapshotRef.state` is `canonical` afterward."

**Errors:**
```typescript
type PublishError =
  | CommonError
  | { kind: "SnapshotNotFound"; snapshot_id: SnapshotId }
  | { kind: "AlreadyCanonical"; snapshot_id: SnapshotId }
```

---

### `discard`

```typescript
function discard(snapshot_id: SnapshotId): Promise<void>
```

Mark a working snapshot as discarded. Eligible for garbage collection by the impl.

**Errors:** `CommonError | { kind: "SnapshotNotFound"; snapshot_id: SnapshotId } | { kind: "Conflict"; reason: string }` (cannot discard canonical).

---

### `list_snapshots`

```typescript
function list_snapshots(filter?: SnapshotFilter): Promise<SnapshotRef[]>

interface SnapshotFilter {
  state?:      ("working" | "canonical" | "discarded")[]
  since?:      string  // ISO 8601
  parent_of?:  SnapshotId
}
```

**Errors:** `CommonError`.

---

## Request entry points

### `ingest`

```typescript
function ingest(fragment: NormalizedFragment): Promise<IngestSummary>

interface IngestSummary {
  ingest_id:           IngestId
  fragment_id:         FragmentId
  accepted_by_filter:  boolean
  atoms:               ProposeResult       // from Atoms (see atoms.md)
  blast?:              BlastReport         // present if a decision atom triggered Blast Radius
  projection:          ReRenderResult      // affected sections re-rendered
}
```

Routes through Ingestion → Atoms → (Blast Radius if a decision atom + wired) → Projection. Returns a composite summary; details from each capability container preserve their original shape.

**Errors:** union of `AcceptError | ProposeError | ReRenderError | …` per called capability.

**Idempotency:** idempotent on `fragment.message_id`.

**Concurrency:** serialized per snapshot.

---

### `query`

```typescript
function query(intent: string, hints?: QueryHints): Promise<QueryResult>

interface QueryHints {
  generator?:     string         // override generator selection
  audience?:      "pm" | "designer" | "engineer" | "exec"
  scope_hint?:    ScopePath
}

interface QueryResult {
  response:           string             // markdown / structured content
  generator_used:     string
  capabilities_used:  string[]           // ["hierarchical_nav", "blast_radius", ...]
  provenance: {
    atoms:           AtomId[]
    navigation:      string[]            // path segments consulted
    sources:         string[]            // external sources consulted
  }
  confidence:         number             // 0.0–1.0
  faithfulness?:      "ok" | "warning" | "failed"
}
```

Routes to [Synthesis](./synthesis.md) if wired; falls back to direct Atoms reads if not.

**Errors:** `CommonError | { kind: "NoGenerator"; intent: string }`.

---

### `re_render`

```typescript
function re_render(scope?: ScopePath): Promise<ReRenderResult>
```

Explicit projection re-render trigger. See [`projection.md`](./projection.md).

---

### `record_resolution`

```typescript
function record_resolution(
  blast_id:    BlastId,
  atom_id:     AtomId,
  resolution:  BlastResolution,
  rationale:   string
): Promise<BlastReport>
```

Forward a resolution from the agent (or external caller) into Blast Radius. See [`blast-radius.md`](./blast-radius.md) for `BlastResolution`.

**Errors:** `CommonError | { kind: "BlastRadiusNotWired" }`.

---

## Capability discovery

### `capabilities`

```typescript
function capabilities(): Promise<CapabilityInfo[]>

interface CapabilityInfo {
  id:        string    // "ingestion" | "atoms" | "projection" | "orchestration" | "hierarchical_nav" | "blast_radius" | "synthesis" | "llm_gateway" | "quality"
  version:   string    // "aala-v1"
  optional:  boolean
  wired:     boolean
}
```

Lists which capabilities are wired in this deployment. Consumed by Synthesis and clients to reason about graceful degradation.

**Errors:** `CommonError`.

---

## `changes_since`

Shared signature. Event variants (snapshot lifecycle):

```typescript
type OrchestrationEvent =
  | { kind: "SnapshotForked";    payload: { snapshot_id: SnapshotId; parent?: SnapshotId } }
  | { kind: "SnapshotSwitched";  payload: { from?: SnapshotId; to: SnapshotId } }
  | { kind: "SnapshotPublished"; payload: { snapshot_id: SnapshotId } }
  | { kind: "SnapshotDiscarded"; payload: { snapshot_id: SnapshotId } }
```

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new fields in `QueryHints`, new `audience` values, new event kinds, new `CapabilityInfo` fields.

## Notes for implementers

- Snapshot semantics are conceptual; specific realizations (git working tree, versioning DB, content-addressed manifest) are implementation choice. The contract is "the returned `SnapshotRef` identifies a snapshot that downstream capabilities can read/write against coherently."
- `ingest` is synchronous in the contract; an impl may expose an async variant in addition but must support this synchronous call.
- Concurrency on cross-snapshot operations: `switch_to` followed by reads must observe a coherent view. Implementations either block reads during switch or use snapshot-isolated reads.
