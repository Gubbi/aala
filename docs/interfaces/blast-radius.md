# Blast Radius — Interface Contract

**Version:** `aala-v1`
**Optionality:** optional
**Container:** [`L2/07-blast-radius.md`](../L2/07-blast-radius.md) · [`L3/07-blast-radius.md`](../L3/07-blast-radius.md)

Blast Radius computes the set of atoms affected by a sweeping decision and tracks resolution as humans work through it.

## Types

```typescript
interface BlastReport {
  blast_id:           BlastId
  decision_atom_id:   AtomId
  state:              "open" | "in_progress" | "closed"
  opened_at:          string
  closed_at?:         string
  affected:           AffectedAtom[]
  iterations:         number
}

interface AffectedAtom {
  atom_id:            AtomId
  detection:          "direct" | "premise_tag" | "llm_implicit"
  suggested_action:   BlastResolution
  rationale:          string
  resolution?:        BlastResolutionRecord
}

type BlastResolution =
  | "Keep"
  | "Adapt"
  | "Deprecate"
  | "Remove"
  | "Defer"

interface BlastResolutionRecord {
  resolution:      BlastResolution
  rationale:       string
  resolved_at:     string
  resolved_by?:    string   // identity
}

interface BlastOptions {
  pipeline_depth?:   "direct" | "direct_plus_premise" | "full"
  iteration_policy?: "single_pass" | "fixed" | "until_fixed_point"
  max_iterations?:   number
}
```

## Methods

### `analyze`

```typescript
function analyze(
  decision_atom_id:  AtomId,
  options?:          BlastOptions
): Promise<BlastReport>
```

Run the pipeline for a decision atom. Produces a blast report, persists it in the current snapshot via Report Store, flips affected atoms to `Hanging` through Atoms's `update_status` API.

**Errors:**
```typescript
type AnalyzeError =
  | CommonError
  | { kind: "NotDecisionAtom"; atom_id: AtomId }
  | { kind: "AnalysisFailed"; stage: "direct" | "premise" | "implicit"; reason: string }
```

**Idempotency:** idempotent on `decision_atom_id` within a snapshot. Re-running produces the same report (modulo non-determinism in the LLM-implicit pass, which is acknowledged but bounded).

**Concurrency:** serialized per decision atom.

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

Capture a human's decision on an affected atom. Translates the resolution into the appropriate call against Atoms:

| `resolution` | Atoms call |
|---|---|
| `Keep` | `update_status(atom_id, "Active", rationale)` |
| `Adapt` | (caller separately invokes `Atoms.propose(...)` with revised content; Iterative Refiner observes the resulting `Updated` event) |
| `Deprecate` | `update_status(atom_id, "Deprecated", rationale)` |
| `Remove` | `update_status(atom_id, "Removed", rationale, { blast_id })` |
| `Defer` | `update_status(atom_id, "Deferred", rationale)` |

Optionally re-runs the LLM-implicit pass with the new resolution as signal (configured by `iteration_policy`).

**Errors:**
```typescript
type RecordResolutionError =
  | CommonError
  | { kind: "BlastNotFound"; blast_id: BlastId }
  | { kind: "BlastClosed"; blast_id: BlastId }
  | { kind: "AtomNotAffected"; blast_id: BlastId; atom_id: AtomId }
```

**Idempotency:** idempotent on `(blast_id, atom_id)`. Last resolution wins; re-resolving with the same `resolution + rationale` is a no-op.

**Concurrency:** serialized per `(blast_id, atom_id)`. Different atoms in the same blast may be resolved concurrently.

---

### `get_report`

```typescript
function get_report(blast_id: BlastId): Promise<BlastReport | null>
```

Fetch a full blast report.

**Errors:** `CommonError`.

---

### `list_reports`

```typescript
function list_reports(filter?: ReportFilter): Promise<BlastReport[]>

interface ReportFilter {
  state?:           ("open" | "in_progress" | "closed")[]
  opened_since?:    string
  decision_atom?:   AtomId
}
```

**Errors:** `CommonError`.

---

### `unresolved`

```typescript
function unresolved(blast_id?: BlastId): Promise<AffectedAtom[]>
```

Atoms still awaiting human resolution. With no `blast_id`, returns across all open reports.

**Errors:** `CommonError`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type BlastRadiusEvent =
  | { kind: "ReportOpened";          payload: { blast_id: BlastId; decision_atom_id: AtomId; affected_count: number } }
  | { kind: "AffectedAtomResolved";  payload: { blast_id: BlastId; atom_id: AtomId; resolution: BlastResolution } }
  | { kind: "ReportRefined";         payload: { blast_id: BlastId; iteration: number; added: AtomId[]; removed: AtomId[] } }
  | { kind: "ReportClosed";          payload: { blast_id: BlastId; resolutions: Record<BlastResolution, number> } }
```

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new `pipeline_depth` / `iteration_policy` values, new event kinds.

## Notes for implementers

- Pipeline depth is a deployment configuration that affects cost/recall trade-offs; the contract is "given depth D, find at least all atoms reachable by D's mechanism."
- LLM-implicit pass uses LLM Gateway with `aala.blast_implicit`.
- Iterative refinement uses each resolution as signal — the implicit pass may revise its remaining suggestions on subsequent calls.
