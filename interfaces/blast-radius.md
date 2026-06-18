# Blast Radius — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** optional

Blast Radius computes the set of atoms affected by a proposed transition and tracks resolution as humans work through the report. `analyze` and the read methods are read-only; `record_resolution` mutates Blast Radius's own report store, and the atom transitions it drives flow exclusively through `Atoms.update_status` — atom state never changes through any other path. Pipeline and report semantics — stages, the structural-equivalence invariant, iteration, report lifecycle, determinism, retention — are normative in [08 — Blast Radius](../spec/08-blast-radius.md).

## Types

```typescript
interface BlastInput {
  atom_id:         AtomId
  hypothetical:    AtomStatus           // the proposed state to evaluate the cascade for
  depth_limit?:    number               // cap on traversal depth (default: unbounded)
  scope_filter?:   TreeId[]             // restrict the report to listed trees
}

interface BlastReport {
  blast_id:        BlastId
  origin:          AtomId
  origin_state:    AtomStatus           // the hypothetical state the analysis evaluated
  state:           "open" | "in_progress" | "closed"   // lifecycle in spec/08 § Report lifecycle
  opened_at:       string               // RFC 3339
  closed_at?:      string
  impacts:         BlastImpact[]        // structural impact set — from Atoms.simulate_transition
  derived_loss:    AtomId[]             // derived atoms that would be invalidated (cascade channel 3)
  cross_tree_advisories: CrossTreeAdvisory[]  // equivalence-linked twins of impacted atoms — advisory, outside the structural set
  implicit_impacts: ImplicitImpact[]    // LLM-inferred semantic impacts — advisory, outside the structural set; empty when the pass is not wired
  below_threshold?: AtomId[]            // atoms excluded by the composition confidence gate (spec/08 § Confidence gating)
  iterations:      number               // advisory-refinement rounds applied so far (spec/08 § Iteration)
}

interface CrossTreeAdvisory {
  atom_id:         AtomId               // the equivalence-linked twin (in another tree)
  equivalent_to:   AtomId               // the structurally impacted atom it corresponds to
  via:             AtomId               // the kind=equivalence RelationAtom
  tree:            TreeId               // the twin's tree
}

interface BlastImpact {
  atom_id:               AtomId
  predicted_state:       AtomStatus            // what the cascade rules would set it to (hanging, under the current cascade registry — spec/08 § Resolution)
  via:                   CascadeStep[]         // path(s) from the origin — shared shape (00-shared-types.md)
  tree:                  TreeId                // the impacted atom's Scope.tree
  resolution_required:   boolean
  resolution?:           BlastResolutionRecord
}

interface ImplicitImpact {
  atom_id:         AtomId
  explanation:     string               // the model's stated basis for the inferred impact
  tree:            TreeId               // the atom's Scope.tree
}

interface BlastResolutionRecord {
  resolution:      BlastResolution             // the reviewer's choice — closed vocabulary (§ record_resolution)
  rationale:       string
  resolved_at:     string                      // RFC 3339
  resolved_by?:    string                      // identity
}
```

A stored `BlastResolutionRecord` always denotes a **completed** atom transition — the record is written only after its mapped `Atoms.update_status` call has completed (hand-off contract in [§ `record_resolution`](#record_resolution)). There is no pending or partially-applied resolution state.

`CascadeStep` is the shared cascade-path shape ([00 — Shared Types § Cascade paths](./00-shared-types.md#cascade-paths)) — produced by [`Atoms.simulate_transition`](./atoms.md#simulate_transition) and embedded here unchanged.

## Methods

### `analyze`

```typescript
function analyze(input: BlastInput): Promise<BlastReport>
```

Compute the impact of a proposed transition: the analysis treats `input.atom_id` as transitioning to `input.hypothetical`, without applying anything. Read-only — `analyze` never transitions an atom.

The **structural** layer (`impacts` + `derived_loss`) is obtained from [`Atoms.simulate_transition`](./atoms.md#simulate_transition) — the read-only dry-run of the one cascade engine (three channels + composition confidence gate, to fixpoint). Blast Radius does **not** re-implement the traversal; it maps the simulation into the report, collects `cross_tree_advisories` (the `kind=equivalence`-linked twins of impacted atoms — advisory; equivalence transmits no cascade), runs the optional LLM-implicit pass into `implicit_impacts` (advisory), applies `scope_filter`, and stores the report. Stage semantics and the structural-equivalence invariant are normative in [08 — Blast Radius § Pipeline stages](../spec/08-blast-radius.md#pipeline-stages).

**Errors:** `analyzing_impact` — `not_found` (unknown origin atom), `invalid_input` (malformed analysis request — invalid hypothetical state, depth limit, or scope filter).

**Access:** `review` — analysis opens a report, the review workflow's artifact ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

**Idempotency:** idempotent on the input tuple `(atom_id, hypothetical, depth_limit, scope_filter)` while atom state is unchanged — a replay returns the existing open report: same `blast_id`, no second `ReportOpened`. Once atom state has changed since generation, a re-run is a new request and opens a fresh report ([08 § Determinism and report identity](../spec/08-blast-radius.md#determinism-and-report-identity)). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** read-only, concurrent-safe.

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

type BlastResolution =
  | "active"        // restore — the atom stands unchanged; the hang was assessed and cleared
  | "adapted"       // the atom was revised first (Adapt), then restored to active
  | "hanging"       // confirm the hang — the atom stays parked for later review
  | "deferred"
  | "deprecated"
  | "superseded"    // requires meta.superseded_by
  | "duplicate"     // requires meta.duplicate_of
  | "removed"

interface BlastResolutionMeta {
  superseded_by?:  AtomId            // REQUIRED when resolution = "superseded"
  duplicate_of?:   AtomId            // REQUIRED when resolution = "duplicate"
}
// the forwarded subset of StatusOutcomeMeta (atoms.md § update_status); record_resolution
// stamps `blast_id` into the forwarded meta itself — callers never pass it

interface RecordResolutionResult {
  blast_id:              BlastId
  atom_id:               AtomId
  record:                BlastResolutionRecord              // the stored record
  report_state:          "open" | "in_progress" | "closed"  // the report's state after this resolution
  remaining_unresolved:  number                             // impacts in this report still awaiting resolution
  cascaded:              AtomCascadeRecord[]                // atoms the resolution's own transition cascaded to — forwarded from the underlying UpdateStatusResult (atoms.md)
}
```

Record a reviewer's resolution for one affected atom. `BlastResolution` is a **closed** vocabulary: every value it admits has a defined mapping below, and nothing else is accepted — `rejected` is unrepresentable because a committed atom cannot be rejected ([spec/07 § Transition table](../spec/07-atom-lifecycle.md#transition-table)). Each resolution maps to exactly one `Atoms.update_status` call, with `blast_id` stamped into the forwarded meta on every row so the resolution is auditable on the atom itself (`status.meta.blast_id`):

| `resolution` | Atoms call |
|---|---|
| `"active"` | `update_status(atom_id, "active", rationale, { blast_id })` — restoring a hanging atom unchanged |
| `"adapted"` | `update_status(atom_id, "active", rationale, { blast_id })` — the explicit `hanging → active` step of Adapt; the content revision precedes this call (below) |
| `"hanging"` | `update_status(atom_id, "hanging", rationale, { blast_id })` — acknowledging the hang; a no-op replay when the atom already holds it ([spec/07 § API for transitions](../spec/07-atom-lifecycle.md#api-for-transitions)) |
| `"deferred"` | `update_status(atom_id, "deferred", rationale, { blast_id })` |
| `"deprecated"` | `update_status(atom_id, "deprecated", rationale, { blast_id })` |
| `"superseded"` | `update_status(atom_id, "superseded", rationale, { blast_id, superseded_by })` — requires `meta.superseded_by` |
| `"duplicate"` | `update_status(atom_id, "duplicate", rationale, { blast_id, duplicate_of })` — requires `meta.duplicate_of` |
| `"removed"` | `update_status(atom_id, "removed", rationale, { blast_id })` |

**Adapt.** `"adapted"` records the Adapt path ([spec/07 § Adapt](../spec/07-atom-lifecycle.md#adapt--special-case)): the caller first revises the atom's content in place — `Atoms.update(...)`, or an `Atoms.propose(...)` the pipeline folds in as a revision — then calls `record_resolution(..., "adapted", ...)`, which issues the explicit `hanging → active` transition and stores the record. The report thereby distinguishes *restored unchanged* (`"active"`) from *revised to accommodate the change* (`"adapted"`). When the resolving content is a **different claim**, it enters as a new atom via `propose` and the hanging atom is resolved with the row matching its exit — `"superseded"`, `"deprecated"`, or `"removed"` — per [spec/07 § Revision, succession, and the prior atom's status](../spec/07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status). The revision events themselves (`Updated` / `Added` / `StatusChanged`) flow through the Atoms Delta Stream as usual; the report is updated by this method, never by event observation.

**Hand-off and atomicity.** The mapped `Atoms.update_status` call executes synchronously within `record_resolution`, after the report-level checks (the report exists and is open, the atom is in the structural set, the resolution request is well-formed) and **before** anything is recorded — apply-then-record:

- **The transition fails** → the propagated `transitioning` pair is the call's outcome. No resolution record is stored, the report's state and counts are unchanged, no event is emitted — a failed call leaves the report exactly as it was.
- **The transition completes** → the resolution record, the report-state transition it triggers (`open → in_progress` on the first record, `→ closed` on the last — [08 § Report lifecycle](../spec/08-blast-radius.md#report-lifecycle)), and the `ImpactResolved` / `ReportClosed` emissions commit together.
- **The record commit fails after the transition completed** (an infra failure between the two stores) → the call fails with the infra end-state; the atom transition stands — it is an ordinary committed Atoms write. The caller retries `record_resolution`: the re-issued `update_status` replays as a no-op (a call naming the atom's current state — its [idempotency row](./00-shared-types.md#idempotency)) and the record commits. The net effect of any number of retries is one transition and one record.

Callers wanting the full report after a resolution fetch it with `get_report`; stream consumers are notified by `ImpactResolved`.

**Errors:** `analyzing_impact` — `not_found` (unknown blast id), `report_closed` (the report is closed), `not_affected` (the atom is not in the report's impact set), `invalid_input` (malformed resolution request — an unrecognized `resolution` value or meta field). The `transitioning` pairs raised by the underlying `Atoms.update_status` (`not_found`, `invalid_transition`, `missing_meta` — covering an absent `superseded_by` / `duplicate_of`) propagate to the caller unchanged, per [10 — Error Model](../spec/10-error-model.md) § Cross-container propagation.

**Access:** `review`.

**Idempotency:** state-equality per `(blast_id, atom_id)` — re-recording the already-recorded `resolution` + `rationale` + `meta` is a full no-op. The replay check **precedes** the report-level checks (mirroring [spec/07 § API for transitions](../spec/07-atom-lifecycle.md#api-for-transitions)): re-acknowledging the final resolution of a report that has since closed is a no-op success, never `report_closed`. A differing re-resolution is not a replay: it overwrites under **last-write-wins** per `(blast_id, atom_id)` — a genuine write that mutates the report, issues the newly mapped `update_status`, and emits `ImpactResolved` ([01 — Conventions § Idempotency and last-write-wins](../spec/01-conventions.md#idempotency-and-last-write-wins)). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per `(blast_id, atom_id)`. Different atoms in the same blast may be resolved concurrently; the underlying transitions remain ordinary writes under the snapshot's write serialization ([00 — Shared Types § Concurrency](./00-shared-types.md#concurrency)).

---

### `get_report`

```typescript
function get_report(blast_id: BlastId): Promise<BlastReport | null>
```

Fetch a full blast report.

**Errors:** `querying` — universal end-states only (absence is `null`, not an error).

**Access:** `query`.

---

### `list_reports`

```typescript
function list_reports(filter?: ReportFilter, page?: PageRequest): Promise<Page<BlastReportSummary>>

interface ReportFilter {
  state?:           ("open" | "in_progress" | "closed")[]
  opened_since?:    string
  origin?:          AtomId
}

interface BlastReportSummary {
  blast_id:          BlastId
  origin:            AtomId
  origin_state:      AtomStatus
  state:             "open" | "in_progress" | "closed"
  opened_at:         string               // RFC 3339
  closed_at?:        string
  impact_count:      number               // |impacts|
  unresolved_count:  number               // impacts without a recorded resolution
  iterations:        number
}
```

Enumerates report **summaries** — the bounded header of each report; the full body (impact lists, advisory layers) is fetched per report with `get_report`. Paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `invalid_query` (malformed filter, or invalid paging cursor).

**Access:** `query`.

---

### `unresolved`

```typescript
function unresolved(blast_id?: BlastId, page?: PageRequest): Promise<Page<UnresolvedImpact>>

interface UnresolvedImpact {
  blast_id:  BlastId                  // the report the impact belongs to
  impact:    BlastImpact
}
```

Impacts still awaiting reviewer resolution. With no `blast_id`, enumerates across all open reports. Every entry carries the `blast_id` of its report, so each is directly actionable: `record_resolution(entry.blast_id, entry.impact.atom_id, …)`. Paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `not_found` (unknown blast id), `invalid_query` (invalid paging cursor).

**Access:** `query`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type BlastRadiusEvent =
  | { kind: "ReportOpened";        payload: { blast_id: BlastId; origin: AtomId; impact_count: number } }
  | { kind: "ImpactResolved";      payload: { blast_id: BlastId; atom_id: AtomId; resolution: BlastResolution } }   // emitted only for a completed resolution — the mapped transition has been applied (§ record_resolution)
  | { kind: "ReportRefined";       payload: { blast_id: BlastId; iteration: number; added: AtomId[]; removed: AtomId[] } }   // added/removed are implicit_impacts entries — the advisory layer (spec/08 § Iteration)
  | { kind: "ReportClosed";        payload: { blast_id: BlastId; resolutions: Partial<Record<BlastResolution, number>> } }   // count of impacts per recorded resolution
```

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible: new `BlastInput` fields, new event kinds, new optional fields on `BlastImpact`, new `BlastResolution` variants.

## Notes for implementers

- `analyze` is read-only and consumes a consistent view of the session's bound snapshot ([00 — Shared Types § Concurrency](./00-shared-types.md#concurrency)).
- The optional LLM-implicit pass calls LLM Gateway with `aala.blast_implicit` to surface non-structural impacts (e.g., "decision X invalidates atom Y's implicit assumptions"). Its results land in `implicit_impacts`, never in `impacts`.
- Refinement of the implicit layer runs asynchronously and is bounded — semantics in [08 § Iteration](../spec/08-blast-radius.md#iteration). It never blocks a `record_resolution` response.
- Report retention is normative in [08 § Storage and retention](../spec/08-blast-radius.md#storage-and-retention).
