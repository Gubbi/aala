# Quality — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** optional (cross-cutting measurement)

Quality measures aala's own LLM pipelines and offers faithfulness verification as a service. Its evaluation targets are the judged surfaces aala itself runs — **extraction** (`aala.extraction`), **conflict classification** (`aala.conflict_judge`), the **blast-implicit pass** (`aala.blast_implicit`), and the **faithfulness judge** (`aala.faithfulness_judge`) — exercised through golden sets, benchmarks, shadow evaluation, and telemetry. Its one outward-facing service, [`verify_faithfulness`](#verify_faithfulness), judges externally generated prose against the atom store. Quality is observation-only — it consumes other containers' read APIs and the LLM Gateway's structured call records, and produces signal. Answer generation and Q&A are not aala concerns ([02 — Conformance § Required facet — Projection](../spec/02-conformance.md#required-facet--projection)); Quality judges prose, it never produces it.

## Types

```typescript
type GoldenSetId      = string  // record-local key, deployer-chosen
type BenchmarkVersion = string  // record-local key, deployer-chosen

// CallId ("call:<opaque>") is a shared identifier, minted by the LLM Gateway one per executed call
// (00 — Shared Types § Identifiers; Appendix A § Identifier minting)

type EvaluatedPipeline = "extraction" | "conflict" | "blast_implicit" | "faithfulness"

type CaseVerdict = "pass" | "warning" | "fail"
// the single verdict vocabulary — used by case results, aggregates, faithfulness reports, and events
```

### Golden cases

A golden case targets exactly one evaluated pipeline and carries that pipeline's input in the same shape the pipeline consumes in production, plus the expectation a correct run satisfies.

```typescript
type GoldenCase =
  | ExtractionCase
  | ConflictCase
  | BlastImplicitCase
  | FaithfulnessCase

interface ExtractionCase {
  case_id:          string
  pipeline:         "extraction"
  fragment:         NormalizedFragment   // replayed through `aala.extraction`
  expected_claims:  ExpectedClaim[]      // what a correct extraction yields
}

interface ExpectedClaim {
  summary:          string      // the claim in words — matched semantically against extracted atoms, never byte-wise
  type?:            AtomType    // when stated, a matching extracted atom MUST have this type
  classification?:  string      // when stated, a matching extracted atom MUST classify under this subject or alias
}

interface ConflictCase {
  case_id:          string
  pipeline:         "conflict"
  proposed:         Atom                 // fixture — see the fixture rules below
  existing:         Atom                 // the canonical candidate the proposal is judged against
  expected_outcome: ConflictOutcomeKind  // enum in interfaces/atoms.md; registry in spec Appendix A § Conflict outcomes
}

interface BlastImplicitCase {
  case_id:          string
  pipeline:         "blast_implicit"
  premise:          Atom                 // the changed atom, as a fixture
  hypothetical:     AtomStatus           // its hypothetical new state
  candidate:        Atom                 // the atom judged for implicit impact
  expected_verdict: "affected" | "not_affected"
}

interface FaithfulnessCase {
  case_id:          string
  pipeline:         "faithfulness"
  text:             string               // the prose to judge
  against:          Atom[]               // the reference set, as fixtures
  expected_verdict: CaseVerdict
}
```

**Fixture rules.** Fixture atoms are complete `Atom` values whose `id`s are case-local labels: they are never resolved against the store and never committed. Fixture `Scope.tree` values are taken at face value — same-tree vs. cross-tree relationships *between a case's fixtures* are preserved for outcome classification — and need not name registered trees.

```typescript
interface GoldenSet {
  set_id:        GoldenSetId
  display_name:  string
  description:   string
  cases:         GoldenCase[]
}

interface GoldenCaseResult {
  case_id:    string
  pipeline:   EvaluatedPipeline
  verdict:    CaseVerdict
  expected:   string                    // the case's expectation, rendered for review
  observed:   string                    // what the pipeline produced, rendered for review
  scores?:    Record<string, number>    // pipeline-specific numerics (e.g., extraction precision / recall over expected_claims)
}

interface GoldenSetResult {
  set_id:       GoldenSetId
  run_at:       string                  // RFC 3339 (00 — Shared Types § Timestamps and durations)
  snapshot_id:  SnapshotId
  cases:        GoldenCaseResult[]
  aggregate: {                          // each count == the number of cases with that verdict; nothing else feeds the buckets
    pass:       number
    warning:    number
    fail:       number
  }
}

interface BenchmarkResult {
  version:      BenchmarkVersion
  run_at:       string                  // RFC 3339
  aggregate:    Record<string, number>  // per evaluated dimension
  regressions:  string[]                // human-readable regression descriptions
  pass:         boolean
}
```

### Faithfulness verification

```typescript
interface FaithfulnessSubject {
  scope?:     ScopePath   // judge against the present-state atoms in this scope subtree
  atom_ids?:  AtomId[]    // judge against exactly these atoms, as they stand
}                          // exactly one of the two MUST be present

interface FaithfulnessReport {
  verdict:      CaseVerdict
  atoms:        AtomId[]            // the resolved reference set the text was judged against
  missed:       MissedClaim[]       // comprehensiveness: reference claims the text does not carry
  distorted:    DistortedClaim[]    // text passages that misstate a reference claim
  unsupported:  UnsupportedClaim[]  // text passages no reference claim supports
}

interface MissedClaim {
  atom_id:  AtomId    // the reference atom whose claim is absent from the text
  summary:  string    // the claim, in words
}

interface DistortedClaim {
  atom_id:  AtomId    // the reference atom the text misstates
  excerpt:  string    // the text passage carrying the distortion
  summary:  string    // how the passage diverges from the claim
}

interface UnsupportedClaim {
  excerpt:  string    // a text passage asserting what no reference atom supports
  summary:  string
}
```

### Feedback and telemetry

```typescript
interface FeedbackEntry {
  call_id:      CallId
  use_case:     UseCaseKey     // the gateway use case behind the judged output, from the call record
  atoms:        AtomId[]       // the call's working atom set, from the call record's context ids ([] when the use case has none)
  fragment?:    FragmentId     // the call's input fragment, for extraction calls
  feedback:     FeedbackInput
  captured_at:  string         // RFC 3339
}

interface FeedbackInput {
  kind:         "thumbs_down" | "structured"
  rationale?:   string
  excerpt?:     string                     // the disputed passage of the judged output, as the recorder saw it —
                                           // recorder-supplied; call records carry no raw content (spec/09 § No content persistence)
  structured?:  Record<string, unknown>
}

interface MetricsQuery {
  timeframe?:   { since: string; until?: string }
  by?:          ("container" | "use_case" | "snapshot" | "tree")[]
  metrics?:     ("latency_p50" | "latency_p99" | "calls" | "errors" | "tokens" | "conflict_rate")[]
}

interface MetricsResult {
  buckets:      MetricBucket[]
}

interface MetricBucket {
  dimensions:   Record<string, string>      // e.g., { container: "atoms", use_case: "aala.extraction" }
  values:       Record<string, number>      // e.g., { latency_p50: 320, calls: 1842 }
}

interface ShadowEvalStatus {
  enabled:      boolean
  use_cases:    UseCaseKey[]   // the keys being sampled (empty when disabled)
  sample_rate:  number         // 0.0–1.0
}
```

## Methods

### `metrics`

```typescript
function metrics(query?: MetricsQuery): Promise<MetricsResult>
```

Aggregate telemetry across containers / use cases / timeframes / trees. Built from the structured call records emitted by LLM Gateway ([09 — LLM Gateway § Observability](../spec/09-llm-gateway.md#observability)) and container Delta Streams.

**Errors:** `evaluating` — `invalid_input` (malformed metrics query).

**Access:** `query` ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

---

### `feedback_archive`

```typescript
function feedback_archive(filter?: FeedbackFilter, page?: PageRequest): Promise<Page<FeedbackEntry>>

interface FeedbackFilter {
  since?:     string       // RFC 3339
  use_case?:  UseCaseKey
  tree?:      TreeId       // entries whose working atom set touches this tree
}
```

The archive of negative feedback, each entry joined to the LLM Gateway call record it judges, paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads). Per-deployment data — never crosses deployment boundaries.

**Errors:** `evaluating` — `invalid_input` (malformed filter, or invalid paging cursor).

**Access:** `query`.

---

### `list_golden_sets`

```typescript
function list_golden_sets(): Promise<GoldenSet[]>
```

The discovery read for golden sets. Bounded by design — golden sets are a deployer-curated registry, not knowledge-base-sized data ([00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads)). Golden sets and benchmark versions are deployment configuration: the deployer provisions them; [`add_golden_case`](#add_golden_case) is the one in-contract extension point, and set creation, case removal, and the benchmark-version registry are deployment provisioning outside the contract.

**Errors:** `evaluating` — universal end-states only.

**Access:** `query`.

---

### `golden_set_results`

```typescript
function golden_set_results(
  set_id:     GoldenSetId,
  snapshot?:  SnapshotId    // a key into archived run records — not a snapshot binding (04 — Snapshots § Cross-snapshot reads)
): Promise<GoldenSetResult | null>
```

Returns the last golden-set run (or a specific snapshot's run, when impl persists per-snapshot results). The `snapshot` argument selects which archived record to return; it does not select the snapshot the call executes against ([04 — Snapshots § Cross-snapshot reads](../spec/04-snapshots.md#cross-snapshot-reads)). Returns `null` if the set has never been run.

**Errors:** `evaluating` — `not_found` (unknown golden-set id).

**Access:** `query`.

---

### `benchmark_results`

```typescript
function benchmark_results(version?: BenchmarkVersion): Promise<BenchmarkResult | null>
```

Last benchmark run, optionally for a specific version; `version` omitted addresses the most recently run version. Returns `null` if benchmarks haven't been run.

**Errors:** `evaluating` — `not_found` (unknown benchmark version).

**Access:** `query`.

---

### `run_benchmark`

```typescript
function run_benchmark(version?: BenchmarkVersion): Promise<BenchmarkResult>
```

Run a benchmark: a deployer-configured, versioned suite — golden sets plus telemetry assertions — evaluating the internal pipelines under the deployment's current routing. `version` omitted runs the latest configured version. Typically invoked by CI on each release or routing change.

**Errors:** `evaluating` — `not_found` (unknown benchmark version), `model_failure` (a judge call failed after the gateway's retries). A benchmark suite that was never configured surfaces as the universal `evaluating` — `not_wired`.

**Access:** `maintain`.

**Concurrency:** serialized — at most one benchmark run at a time.

---

### `run_golden_set`

```typescript
function run_golden_set(set_id: GoldenSetId): Promise<GoldenSetResult>
```

Execute a named golden set against the calling session's bound snapshot. Each case's input is replayed through the same gateway path production uses — `aala.extraction`, `aala.conflict_judge`, `aala.blast_implicit`, or `aala.faithfulness_judge` per the case's `pipeline` — under the deployment's current routing; classification and predicate names in extraction expectations resolve against the snapshot's registries.

A golden run **MUST NOT** mutate the store: extracted candidates are scored against `expected_claims` and discarded, never proposed; conflict and blast-implicit judgments never transition any atom. Per-case `verdict` grades the expectation: `pass` when met, `fail` when missed, `warning` when met with implementation-judged caveats (surfaced in `observed`). `aggregate` counts case verdicts and nothing else.

**Errors:** `evaluating` — `not_found` (unknown golden-set id), `model_failure` (a judge call failed after the gateway's retries).

**Access:** `maintain`.

**Concurrency:** serialized per set.

---

### `verify_faithfulness`

```typescript
function verify_faithfulness(text: string, subject: FaithfulnessSubject): Promise<FaithfulnessReport>
```

Faithfulness verification as a service: judges externally generated prose against the atom store, through `aala.faithfulness_judge`. The reference set is the subject's resolution at call time — the present-state atoms (`active`, `hanging`, `deferred`, `deprecated`) in the `scope` subtree, or exactly the atoms `atom_ids` names, as they stand (explicit ids are taken as given; status filtering of an explicit set is the caller's choice). The report grades three failure surfaces:

- **`missed`** — reference claims the text does not carry (comprehensiveness),
- **`distorted`** — text passages that misstate a reference claim,
- **`unsupported`** — text passages no reference claim supports.

`verdict` derivation is fixed: `pass` **iff** all three lists are empty; `fail` **iff** `distorted` is non-empty; `warning` otherwise.

The report is computed fresh per call and is not persisted; nothing is recorded against the store or the judged text. The audit trail is the gateway's call records plus the `FaithfulnessVerified` event, which carries counts only — never content.

**Errors:** `evaluating` — `not_found` (an `atom_ids` entry does not resolve in the snapshot), `invalid_input` (empty `text`; a subject with neither or both arms; a malformed scope path; a subject resolving to an empty reference set), `model_failure` (the judge call failed after the gateway's retries).

**Access:** `query`.

**Concurrency:** concurrent-safe.

---

### `record_feedback`

```typescript
function record_feedback(call_id: CallId, feedback: FeedbackInput): Promise<void>
```

End-user-supplied feedback (thumbs / structured complaint) is captured against the originating `call_id` — the LLM Gateway call record behind the output being judged ([Appendix A § Identifier minting](../spec/appendix-a-registries.md#identifier-minting)). Negative feedback lands in the archive read by [`feedback_archive`](#feedback_archive), joined to the call record's use case and context ids.

**Errors:** `evaluating` — `not_found` (unknown call id), `invalid_input` (malformed feedback payload).

**Access:** `review` — recording a verdict on an output ([10 — Error Model § The standardized Tier-1 vocabulary](../spec/10-error-model.md#the-standardized-tier-1-vocabulary)).

**Idempotency:** state-equality per `call_id` — re-recording identical feedback is a full no-op. Differing feedback overwrites under **last-write-wins** per `call_id` ([01 — Conventions § Idempotency and last-write-wins](../spec/01-conventions.md#idempotency-and-last-write-wins)). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

---

### `add_golden_case`

```typescript
function add_golden_case(set_id: GoldenSetId, case_: GoldenCase): Promise<void>
```

Deployer extends a golden set with a new case. The case is validated against its `pipeline` arm's shape.

**Errors:** `evaluating` — `not_found` (unknown golden-set id), `invalid_input` (the case fails structural validation).

**Access:** `maintain`.

**Idempotency:** idempotent on `case_.case_id` (key-identified) — a reused case id is a replay: a full no-op; the recorded case stands. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

---

### `enable_shadow_eval`

```typescript
function enable_shadow_eval(
  use_cases?:   UseCaseKey[],  // default: every evaluated-pipeline key with a configured shadow route
  sample_rate?: number         // 0.0–1.0, default 0.05
): Promise<void>
```

Shadow evaluation samples live internal-pipeline traffic. While enabled, a sampled fraction of gateway calls under the named use-case keys is additionally executed against the deployer-configured **shadow route** for that key — a second model selection, configured alongside the primary routing — and a `ShadowEvalDiff` event is emitted when the two structured outputs disagree. The shadow output is never returned to the caller, never mutates state, and is never cached; primary results are unaffected. The tee point is implementation-internal (the gateway's call path is the natural one); the contract binds the observable behavior — divergences appear as `ShadowEvalDiff` events, and shadow calls appear in `LLMGateway.usage()` under the shadow model.

A key with no shadow route configured is never sampled.

**Errors:** `evaluating` — `invalid_input` (sample rate outside `[0.0, 1.0]`, or a use-case key that is not registered).

**Access:** `maintain`.

**Idempotency:** state-equality — re-applying the current configuration is a full no-op; a differing configuration overwrites under **last-write-wins** (one shadow-eval configuration exists at a time). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

---

### `disable_shadow_eval`

```typescript
function disable_shadow_eval(): Promise<void>
```

Turns shadow evaluation off. A no-op when already disabled.

**Errors:** `evaluating` — universal end-states only.

**Access:** `maintain`.

---

### `shadow_eval_status`

```typescript
function shadow_eval_status(): Promise<ShadowEvalStatus>
```

The current shadow-evaluation configuration.

**Errors:** `evaluating` — universal end-states only.

**Access:** `query`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type QualityEvent =
  | { kind: "BenchmarkRun";         payload: { version: BenchmarkVersion; pass: boolean; result_summary: Record<string, number> } }
  | { kind: "GoldenSetRun";         payload: { set_id: GoldenSetId; pass: number; warning: number; fail: number } }   // CaseVerdict counts
  | { kind: "FeedbackRecorded";     payload: { call_id: CallId; kind: "thumbs_down" | "structured" } }
  | { kind: "FaithfulnessVerified"; payload: { verdict: CaseVerdict; atoms: number; missed: number; distorted: number; unsupported: number } }
      // audit record, applies as a no-op — counts only, never content; nothing is persisted by verify_faithfulness
  | { kind: "ShadowEvalDiff";       payload: { use_case: UseCaseKey; call_id: CallId } }
      // audit record, applies as a no-op — call_id names the primary call's record
```

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible: new event kinds, new fields in result types, new `MetricsQuery` dimensions, new `MetricsQuery` metrics (e.g., `fallback_rate`), new golden-case pipelines (each clearly documented).

## Notes for implementers

- Quality is observation-only. It MUST never mutate any other container's state — golden runs, benchmarks, shadow calls, and faithfulness verifications read, judge, and discard.
- External agents that compose prose over aala's read surfaces submit it to [`verify_faithfulness`](#verify_faithfulness) for grounding checks — the same judge (`aala.faithfulness_judge`) that faithfulness golden cases evaluate. Quality offers the judgment; it never owns or invokes any external generation path.
- Telemetry records and aggregates are deployment-internal: the contract is scoped to one logical deployment ([02 — Conformance § Single-deployment scope](../spec/02-conformance.md#single-deployment-scope)). Aggregation across deployments is operator composition outside this contract; an operator that aggregates is expected to carry only structural signals across deployment boundaries, never raw content.
- Tree-bucketed metrics enable comparison between trees (e.g., "extraction precision in planning vs implementation").
