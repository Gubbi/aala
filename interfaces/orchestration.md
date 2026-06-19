# Orchestration — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** non-optional

Orchestration is the outside-world entry point. It owns the snapshot lifecycle, the tree registry, and dispatches requests to capability containers.

As the customer touch point, Orchestration is the tier that produces **Operation Errors** (Tier 1, per [10 — Error Model](../spec/10-error-model.md)): it frames a failure in the caller's intent and **delegates classification** from the **Activity Error** (Tier 2) it wraps under `cause`. The shape a call raises is exactly what its **Errors** line declares, and the declarations follow one rule:

- **Operation entry points produce Tier 1.** Every mutating method (`fork`, `switch_snapshot`, `publish_as_canonical`, `discard`, `register_tree`, `update_tree`) and every composing entry point (`ingest`, `record_resolution`) frames the Tier-1 operation it declares around the Activity Error its own work raised or a capability container returned.
- **Introspective reads surface Tier 2 directly.** `current_snapshot`, `list_snapshots`, `list_trees`, and `capabilities` dispatch nothing and read only Orchestration's own state — the session's binding, the snapshot and tree registries, the capability declaration; their `querying` Activity Error is already framed at the caller's altitude, and a Tier-1 wrapper would add a tier that delegates everything and frames nothing.
- **Calls addressed to a capability container's methods** — the read surface under [`query`](#query) included — surface the addressed method's declared Activity Error unchanged. Tier 1 is the edge framing a customer touch point applies, and for these calls the touch point is the product or integration embedding aala ([10 § The three-tier chain](../spec/10-error-model.md#the-three-tier-chain)); no method inside the contract frames them.

Every Operation/Activity Error carries the operation's `correlation_id` as `trace_id`.

Every call executes within a **session**, and a session binds exactly one snapshot at a time — the binding is the sole snapshot selector for every read and write. [04 — Snapshots § Session binding](../spec/04-snapshots.md#session-binding) owns the binding model; the methods below are the session's snapshot-management surface: `fork` creates snapshots, `switch_snapshot` rebinds the calling session, `current_snapshot` reports its binding. How a session is established and conveyed (an HTTP token, an MCP connection, a CLI working directory) is wire-profile- and deployment-specific.

At establishment a session also receives its **access grants** — `(snapshot, operation type)` pairs the perimeter checks on every call admission. Each method declares the operation type its check uses in an **Access** line; the model, the subject-snapshot rule, and the blanket `authorizing` failure pairs are normative in [02 — Conformance § Access control](../spec/02-conformance.md#access-control).

## Snapshot management

### `current_snapshot`

```typescript
function current_snapshot(): Promise<SnapshotRef>
```

Returns the calling session's bound snapshot — the snapshot every read and write this session issues executes against ([04 § Session binding](../spec/04-snapshots.md#session-binding)).

**Errors:** `querying` — universal end-states only.

**Access:** none — session-floor: callable by any established session ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `fork`

```typescript
function fork(from?: SnapshotId): Promise<SnapshotRef>
```

Create a new working snapshot, optionally forked from a specific source snapshot (default: the snapshot the canonical pointer designates). `fork` does **not** rebind the calling session — follow with `switch_snapshot` to work in the new snapshot.

**Errors:** `OperationError` — `operation: "manage_snapshot"`, wrapping `snapshotting` — `not_found` (the `from` parent snapshot does not exist).

**Access:** `query` against the **source** snapshot (`from`, or the canonical pointer's target when omitted) — a read-entry act: the fork makes the source's content readable in the created snapshot. The creating session receives all six operation types on the created snapshot, atomically with the fork ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `switch_snapshot`

```typescript
function switch_snapshot(snapshot_id: SnapshotId): Promise<void>
```

Rebind the **calling session** to the given snapshot ([04 § Session binding](../spec/04-snapshots.md#session-binding)). Other sessions are unaffected. Calls this session admits after the rebind execute against the new binding; calls admitted before it complete against the old one. The target MAY be a `working` or a `canonical` snapshot — the latter for stable historical reads, where write entry points then fail per the session-binding rules. A `changes_since(ref)` cursor held from the previous binding subsequently fails with `stale_ref`, and the consumer re-baselines ([05 — Delta Streams § Cross-snapshot behavior](../spec/05-delta-streams.md#cross-snapshot-behavior)).

**Errors:** `OperationError` — `operation: "manage_snapshot"`, wrapping `snapshotting` — `not_found` (unknown `snapshot_id`), `discarded` (the target snapshot has been discarded).

**Access:** `query` against the **target** snapshot — rebinding makes the target the session's read context; what the session can then do there is checked per call ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `publish_as_canonical`

```typescript
function publish_as_canonical(snapshot_id: SnapshotId): Promise<SnapshotRef>
```

Move the deployment's **canonical pointer** to the named working snapshot by atomic compare-and-swap ([04 — Snapshots § Canonicalization](../spec/04-snapshots.md#canonicalization) owns the semantics). The publish succeeds iff it is a fast-forward — the currently canonical snapshot is an ancestor of the named snapshot (or the pointer is unset). On success the snapshot's state becomes `canonical` and a `SnapshotPublished` event is emitted; the previously canonical snapshot keeps state `canonical` as immutable history — only the pointer moves. If the pointer moved since the snapshot's fork, the call fails with `non_fast_forward`; recovery is re-fork from the new canonical and replay (snapshot merge is out of scope for `aala-v0.1`).

In deployments where canonicalization is externally managed (e.g., the user merging via git), the compare-and-swap is realized through the external mechanism where the implementation can drive it; where it cannot, this method is not wired (universal `not_wired`) and the pointer moves only by external action, observed and emitted as `SnapshotPublished` ([04 § Canonicalization](../spec/04-snapshots.md#canonicalization)).

**Errors:** `OperationError` — `operation: "manage_snapshot"`, wrapping `snapshotting` — `not_found` (unknown `snapshot_id`), `non_fast_forward` (the canonical pointer moved since this snapshot's fork; re-fork and replay), `discarded` (the named snapshot has been discarded).

**Access:** `manage_snapshot` against the snapshot the **canonical pointer currently designates** — the baseline the publish moves the deployment off; when the pointer is unset (first publish), the requirement is deployment-defined ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

**Idempotency:** state-equality at the pointer — publishing the snapshot the canonical pointer already designates is a no-op success returning its `SnapshotRef`. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

---

### `discard`

```typescript
function discard(snapshot_id: SnapshotId): Promise<void>
```

Mark a working snapshot as discarded. Eligible for garbage collection by the implementation. Discarding a snapshot other sessions are bound to is permitted; their subsequent calls fail with `snapshotting` / `discarded` until they rebind ([04 § Session binding](../spec/04-snapshots.md#session-binding)).

**Errors:** `OperationError` — `operation: "manage_snapshot"`, wrapping `snapshotting` — `not_found` (unknown `snapshot_id`), `canonical_protected` (the canonical snapshot cannot be discarded).

**Access:** `manage_snapshot` against the **named** snapshot ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `list_snapshots`

```typescript
function list_snapshots(filter?: SnapshotFilter, page?: PageRequest): Promise<Page<SnapshotRef>>

interface SnapshotFilter {
  state?:      ("working" | "canonical" | "discarded")[]
  since?:      string  // RFC 3339
  parent_of?:  SnapshotId
}
```

Snapshots accumulate over a deployment's lifetime (canonical history, retained discards), so the listing is paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `invalid_query` (malformed filter, or invalid paging cursor).

**Access:** `query`.

---

## Tree management

### `list_trees`

```typescript
function list_trees(): Promise<TreeInfo[]>

interface TreeInfo {
  tree:          TreeId
  display_name:  string
  description?:  string
  atom_count:    number
  created_at:    string
}
```

Returns the trees registered in the current snapshot.

**Errors:** `querying` — universal end-states only.

**Access:** `query`.

---

### `register_tree`

```typescript
function register_tree(
  tree:          TreeId,
  display_name:  string,
  description?:  string
): Promise<TreeInfo>
```

Register a new tree. Initializes its standard-library ClassificationAtoms and PredicateKindAtoms (see [03 — Data Model](../spec/03-data-model.md)).

**Errors:** `OperationError` — `operation: "manage_tree"`, wrapping `managing` — `already_registered` (the tree id is registered with *differing* `display_name` / `description` — a genuine id conflict; an identical re-registration is a replay, below), `invalid_tree` (the tree id is malformed).

**Access:** `manage_tree`.

**Idempotency:** state-equality per `tree` — re-registering an existing tree with the identical `display_name` and `description` is a full no-op returning the existing `TreeInfo`, so a caller retrying a timed-out registration cannot misread its own success as a conflict. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

---

### `update_tree`

```typescript
function update_tree(
  tree:          TreeId,
  display_name?: string,
  description?:  string
): Promise<TreeInfo>
```

Update a tree's display metadata. `TreeId` is **immutable** — the same principle as the stable `AtomId`: it is the reference every `Scope.tree` carries, and no operation rewrites it ([12 — Edge Cases § Tree lifecycle](../spec/12-edge-cases.md#tree-lifecycle)). Only `display_name` and `description` are mutable; an omitted parameter is left unchanged. Emits a `TreeRenamed` event carrying the changed display fields.

**Errors:** `OperationError` — `operation: "manage_tree"`, wrapping `managing` — `not_found` (the tree id is not registered).

**Access:** `manage_tree`.

**Idempotency:** state-equality — re-applying the current display values, or supplying none, is a full no-op (no event is emitted). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** last-write-wins per field — concurrent differing writes are genuine writes, the later value displacing the earlier per field ([01 — Conventions § Idempotency and last-write-wins](../spec/01-conventions.md#idempotency-and-last-write-wins)).

---

## Request entry points

The perimeter assigns a `request:<opaque>` id at admission of **every externally initiated request** other than `ingest` (which already carries its own `ingest:<opaque>` id) — the snapshot and tree mutations above and the dispatched reads under [`query`](#query) included. On a write, that id becomes the `correlation_id` stamped on every downstream change event across containers, per [05 — Delta Streams § Correlation across streams](../spec/05-delta-streams.md#correlation-across-streams) — enabling cross-stream grouping and debugging of one request's full effect. A read emits no change events; its id surfaces only as the `trace_id` of any error it raises and as the `aala.correlation_id` telemetry attribute ([10 — Error Model § `trace_id`](../spec/10-error-model.md#trace_id--the-operations-correlation_id-carried-on-the-error)).

### `ingest`

```typescript
function ingest(fragment: NormalizedFragment, options?: IngestOptions): Promise<IngestSummary>

interface IngestOptions {
  target_tree?:  TreeId             // override default tree assignment
}

interface IngestSummary {
  ingest_id:           IngestId            // from Ingestion's AcceptResult (see ingestion.md)
  accepted_by_filter:  boolean
  fragment_id?:        FragmentId          // present iff accepted_by_filter — rejected fragments are never persisted
  reject_reason?:      string              // present iff accepted_by_filter is false
  atoms?:              ProposeResult       // present iff accepted_by_filter — from Atoms (see atoms.md)
  blast_id?:           BlastId             // present iff this ingest ran the optional side-effect analysis (08 — Blast Radius § When it runs); read the report via BlastRadius.get_report
  projection?:         ProjectionEvent[]   // present iff accepted_by_filter — DocChanged/DocAdded/DocRemoved/IndexUpdated from the host's projection facet (see projection.md)
}
```

Routes through Ingestion → Atoms (which re-derives its own projection facet for affected sections) → (Blast Radius if wired and triggered). Returns a composite summary; details from each capability container preserve their original shape.

A filter rejection is a **successful** call: the pipeline stops at Ingestion, and the summary carries `accepted_by_filter: false` with `reject_reason` — `fragment_id`, `atoms`, and `projection` are absent because nothing was persisted and no downstream container ran ([`Ingestion.accept`](./ingestion.md#accept)). A retry of a rejected `message_id` re-evaluates the relevance filter rather than replaying the rejection, per Ingestion's idempotency contract.

Whether an ingest runs the optional Blast Radius side-effect analysis is implementation-discretionary; the trigger and semantics are owned by [08 — Blast Radius § When it runs](../spec/08-blast-radius.md#when-it-runs). When it ran, `blast_id` names the opened report — the report body is read via [`BlastRadius.get_report`](./blast-radius.md#get_report), never embedded in the summary. A failed side-effect analysis never fails the ingest (its pairs are absent from the Errors declaration below): the summary simply carries no `blast_id`.

**Errors:** `OperationError` — `operation: "ingest"`, wrapping the failing capability's Activity Error unchanged. The reachable `(activity, end_state)` pairs are those declared on [`Ingestion.accept`](./ingestion.md#accept) and [`Atoms.propose`](./atoms.md#propose). Classification is delegated from the wrapped Activity Error.

**Access:** `ingest`.

**Idempotency:** retry-safe by composition on `fragment.message_id` — a retry replays completed per-container steps as no-ops and resumes incomplete ones ([10 — Error Model § Cross-container failures](../spec/10-error-model.md#cross-container-failures)). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per snapshot.

---

### `query`

The read entry point is **dispatch, not composition**: there is no `query` method, no query language, and no composed answer. A read is addressed to a capability container's own read method, with exactly the signature, arguments, result shape, and paging that method's interface file declares; the rpc-class wire profile's method binding is the addressing scheme ([13 — Wire Profiles § Wire profile registry](../spec/13-wire-profiles.md#wire-profile-registry) — `POST <base-url>/<container>/<method>` for `aala-wire/json+http`, the `<container>.<method>` tool for `aala-wire/json+mcp`). Orchestration's perimeter role applies to these calls as to every other and ends at admission: establish the session context, check the **`query`** grant the addressed method declares, mint the `request:` correlation id ([§ Request entry points](#request-entry-points)) — then the addressed method executes unchanged.

The dispatched read surface is every capability-container method whose **Access** line declares `query` — among them:

- **Structured Atoms reads** — `get_by_id`, `get_by_ids`, `traverse_relations`, `list_by_classification`, `list_scope`, `simulate_transition`, … ([`atoms.md`](./atoms.md))
- **Projection-facet reads** — `list(prefix?, page?)`, `read(path)`, `read_index(root?, depth?)` over a host container's own content (always Atoms; optionally Ingestion / Blast Radius / Hierarchical Navigation) ([`projection.md`](./projection.md))
- **Hierarchical Navigation reads** — `axes`, `navigate`, `node_atoms`, `locate` ([`hierarchical-nav.md`](./hierarchical-nav.md))
- **Blast Radius report reads** — `get_report`, `list_reports`, `unresolved` ([`blast-radius.md`](./blast-radius.md))
- **Ingestion provenance reads** — `get`, `list` ([`ingestion.md`](./ingestion.md))
- **Quality measurement reads** — `metrics`, `feedback_archive`, `list_golden_sets`, `golden_set_results`, `benchmark_results`, `shadow_eval_status`, and the `verify_faithfulness` judgment service ([`quality.md`](./quality.md))
- **Delta-Stream reads** — `changes_since` / `as_stream` on every container ([`00-shared-types.md § Delta-Stream events`](./00-shared-types.md#delta-stream-events))

External agents work directly against this surface — navigating the projection index and fetching prose to ground, then issuing structured Atoms reads for precision — and compose answers and documents themselves. Q&A and document generation are not aala concerns; see [`analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md) (informative).

`query` as an `OperationKind` names this surface in the Tier-1 vocabulary ([Appendix A § Operations (Tier 1)](../spec/appendix-a-registries.md#operations-tier-1)): it is the **Access** class every read method declares, and the operation frame an embedding product applies when it presents a failed read in its own vocabulary. Inside the contract, no method frames it — per the rule in the preamble, a dispatched read surfaces the addressed method's Activity Error directly.

**Errors:** the addressed method's declared `(activity, end_state)` pairs, surfaced unchanged — `querying` pairs from Atoms (incl. the Projection facet), Ingestion, and Blast Radius report reads; `navigating` pairs from Hierarchical Navigation; `evaluating` pairs from Quality reads. No `OperationError` is produced.

**Access:** `query` — declared by each dispatched method itself; one grant class covers the read surface.

---

### `record_resolution`

```typescript
function record_resolution(
  blast_id:    BlastId,
  atom_id:     AtomId,
  resolution:  BlastResolution,
  rationale:   string,
  meta?:       BlastResolutionMeta
): Promise<RecordResolutionResult>
```

Forward a reviewer's resolution into Blast Radius unchanged — the resolution vocabulary, meta shape, mapping, and hand-off contract are [`BlastRadius.record_resolution`](./blast-radius.md#record_resolution)'s.

**Errors:** `OperationError` — `operation: "review"`. When Blast Radius is not present in this deployment, the wrapped Activity Error is the universal `analyzing_impact` — `not_wired`; otherwise the Activity Error from [`BlastRadius.record_resolution`](./blast-radius.md#record_resolution) propagates unchanged and is wrapped as-is.

**Access:** `review`.

---

## Capability discovery

### `capabilities`

```typescript
function capabilities(): Promise<OrchestrationCapabilities>

interface OrchestrationCapabilities {
  capabilities:       CapabilityInfo[]    // one entry per container in the registry of 02 — Conformance § Required containers
  projection_facet:   string[]            // container ids whose Projection facet is implemented; always includes "atoms"
  projection_profile: ProfileKey          // the single output profile every projection is materialized under; "okf" by default (Appendix A § Projection output profiles)
  wire_profiles:      string[]            // declared wire profiles from the registry in 13 — Wire Profiles; non-empty
  default_profile?:   string              // implementation's preferred wire profile; caller MAY override
}

interface CapabilityInfo {
  id:        string    // "ingestion" | "atoms" | "orchestration" | "llm_gateway" | "hierarchical_nav" | "blast_radius" | "quality"
  version:   string    // declared interface version, e.g. "aala-v0.1"
  optional:  boolean   // the container's registry status (informative mirror of spec Appendix A § Containers)
  wired:     boolean   // true iff this deployment realizes the container
}
```

This declaration is the normative home of the `OrchestrationCapabilities` shape ([01 — Conventions § Normative ownership](../spec/01-conventions.md#normative-ownership)); the obligation to expose it and its stability rule are normative in [02 — Conformance § Capability declaration](../spec/02-conformance.md#capability-declaration). Consumed by clients and external agents to reason about graceful degradation and by callers to select an interop-compatible wire profile.

The `capabilities` list **MUST** carry exactly one `CapabilityInfo` entry per container in the container registry ([Appendix A § Containers](../spec/appendix-a-registries.md#containers); conformance rules in [02 — Conformance § Required containers](../spec/02-conformance.md#required-containers)) — required and optional alike. Projection is a facet, not a container, and never appears as an entry. A container is **available** exactly when its entry carries `wired: true` — the one predicate callers branch on. Required containers carry `wired: true` in every conformant deployment; a `wired: false` entry declares an optional container this deployment does not realize, and any call to it fails with the universal `not_wired` end-state ([10 — Error Model § Universal end-states](../spec/10-error-model.md#universal-end-states)).

`projection_facet` lists the container ids whose Projection facet is implemented; it **MUST** include `"atoms"` (the facet 02 requires) and **MUST** list every other container whose facet the deployment implements.

`projection_profile` names the **single output profile** every projection in this deployment is materialized under ([`projection.md` § Output profile](./projection.md#output-profile); registry in [Appendix A § Projection output profiles](../spec/appendix-a-registries.md#projection-output-profiles)). It is `"okf"` unless the deployer configured a custom profile. A deployment runs exactly one profile, so this is a single `ProfileKey`, not a list; the projection read methods select no profile per call. Changing it is an operational re-projection ([`projection.md` § Notes for implementers](./projection.md#notes-for-implementers)), so the value is stable for the process lifetime like the rest of this declaration.

`wire_profiles` **MUST** contain at least one entry from the registry in [13 — Wire Profiles](../spec/13-wire-profiles.md). A caller targeting wire-level interop **MUST** pick a profile from this list before issuing other calls.

Example return value:

```typescript
const example: OrchestrationCapabilities = {
  capabilities: [
    { id: "ingestion",        version: "aala-v0.1", optional: false, wired: true  },
    { id: "atoms",            version: "aala-v0.1", optional: false, wired: true  },
    { id: "orchestration",    version: "aala-v0.1", optional: false, wired: true  },
    { id: "llm_gateway",      version: "aala-v0.1", optional: false, wired: true  },
    { id: "hierarchical_nav", version: "aala-v0.1", optional: true,  wired: false },
    { id: "blast_radius",     version: "aala-v0.1", optional: true,  wired: true  },
    { id: "quality",          version: "aala-v0.1", optional: true,  wired: false }
  ],
  projection_facet:   ["atoms", "blast_radius"],
  projection_profile: "okf",
  wire_profiles:      ["aala-wire/json+http", "aala-wire/yaml+git"],
  default_profile:    "aala-wire/json+http"
}
```

**Errors:** `querying` — universal end-states only.

**Access:** none — session-floor: callable by any established session; this is the declaration a caller needs before it can do anything else ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

## `changes_since`

Shared signature. Event variants (snapshot lifecycle + tree lifecycle):

```typescript
type OrchestrationEvent =
  | { kind: "SnapshotForked";    payload: { snapshot_id: SnapshotId; parent?: SnapshotId } }
  | { kind: "SnapshotPublished"; payload: { snapshot_id: SnapshotId; previous?: SnapshotId } }   // the canonical pointer moved to snapshot_id; previous = the snapshot it moved off (absent on the first publish)
  | { kind: "SnapshotDiscarded"; payload: { snapshot_id: SnapshotId } }
  | { kind: "TreeRegistered";    payload: { tree: TreeId; display_name: string } }
  | { kind: "TreeRenamed";       payload: { tree: TreeId; display_name?: string; description?: string } }   // display-level metadata change via update_tree — carries the changed fields; TreeId is immutable, never an id swap
```

Session rebinding (`switch_snapshot`) emits no event: a binding is session-local state, not shared deployment state — the caller that rebinds knows synchronously, and stream consumers holding a stale cursor discover the rebind through the `stale_ref` error contract ([05 — Delta Streams § Cross-snapshot behavior](../spec/05-delta-streams.md#cross-snapshot-behavior)).

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible: new event kinds, new `CapabilityInfo` fields, new `IngestOptions` fields, new `projection_facet` entries, newly registered output profiles, new wire profile entries in the registry.

## Notes for implementers

- Snapshot semantics are conceptual; specific realizations (git working tree, versioning DB, content-addressed manifest) are implementation choice. The contract is "the returned `SnapshotRef` identifies a snapshot that downstream capabilities can read/write against coherently." A session's realization is equally open (HTTP token, MCP connection, CLI working directory); the binding contract in [04 — Snapshots § Session binding](../spec/04-snapshots.md#session-binding) is what conformance tests check.
- `ingest` is synchronous in the contract; an implementation MAY expose an async variant in addition but MUST support this synchronous call.
- `TreeId` is immutable, so no tree-migration operations exist; tree merge / split are out of scope for `aala-v0.1` ([12 — Edge Cases § Tree lifecycle](../spec/12-edge-cases.md#tree-lifecycle)). The tree registry mutations that do exist (`register_tree`, `update_tree`) SHOULD be serialized against each other.
- Concurrency on rebinding: a call executes wholly against the binding in effect at its admission; `switch_snapshot` takes effect for calls admitted after it returns. Implementations either serialize a session's calls around the rebind or use snapshot-isolated reads.
- The `wire_profiles` field on capabilities is mandatory and non-empty. An implementation that wishes to support gRPC, alternate serializations, or future wire profiles declares them here once those profiles are added to the registry.
