# Quality — Interface Contract

**Version:** `aala-v1`
**Optionality:** optional (cross-cutting measurement)
**Container:** [`L2/10-quality.md`](../L2/10-quality.md) · [`L3/10-quality.md`](../L3/10-quality.md)

Quality measures the system's outputs. It runs benchmarks and golden sets, scores responses for retrieval precision and faithfulness, captures failures, aggregates telemetry. It is observation-only — it consumes other containers' read APIs and produces signal.

## Types

```typescript
type GoldenSetId = string
type BenchmarkVersion = string
type CallId = string

interface GoldenCase {
  case_id:              string
  question:             string
  expected_sections:    SectionPath[]
  expected_traits:      string[]    // e.g., "mentions blast radius", "cites adr-0042"
}

interface GoldenSet {
  set_id:               GoldenSetId
  display_name:         string
  description:          string
  cases:                GoldenCase[]
}

interface GoldenCaseResult {
  case_id:              string
  retrieval: {
    precision:          number
    recall:             number
    matched:            SectionPath[]
    missing:            SectionPath[]
    extra:              SectionPath[]
  }
  faithfulness:         "ok" | "warning" | "failed"
  rubric_scores:        Record<string, number>
  response_excerpt:     string
}

interface GoldenSetResult {
  set_id:               GoldenSetId
  run_at:               string
  snapshot_id:          SnapshotId
  cases:                GoldenCaseResult[]
  aggregate: {
    pass:               number
    warning:            number
    fail:               number
  }
}

interface BenchmarkResult {
  version:              BenchmarkVersion
  run_at:               string
  aggregate:            Record<string, number>      // per dimension
  regressions:          string[]                    // human-readable regression descriptions
  pass:                 boolean
}

interface FailedQueryEntry {
  call_id:              CallId
  intent:               string
  response_excerpt:     string
  provenance: {
    atoms:              AtomId[]
    navigation:         string[]
  }
  feedback: {
    kind:               "thumbs_down" | "structured"
    rationale?:         string
    structured?:        Record<string, unknown>
  }
  captured_at:          string
}

interface FeedbackInput {
  kind:                 "thumbs_down" | "structured"
  rationale?:           string
  structured?:          Record<string, unknown>
}

interface MetricsQuery {
  timeframe?:           { since: string; until?: string }
  by?:                  ("container" | "use_case" | "snapshot")[]
  metrics?:             ("latency_p50" | "latency_p99" | "calls" | "errors" | "tokens")[]
}

interface MetricsResult {
  buckets:              MetricBucket[]
}

interface MetricBucket {
  dimensions:           Record<string, string>      // e.g., { container: "atoms", use_case: "aala.extraction" }
  values:               Record<string, number>      // e.g., { latency_p50: 320, calls: 1842 }
}
```

## Methods

### `metrics`

```typescript
function metrics(query?: MetricsQuery): Promise<MetricsResult>
```

Aggregate telemetry across containers / use cases / timeframes. Built from structured call records emitted by LLM Gateway and Change Logs.

**Errors:** `CommonError`.

---

### `failed_queries`

```typescript
function failed_queries(filter?: FailedQueryFilter): Promise<FailedQueryEntry[]>

interface FailedQueryFilter {
  since?:             string
  intent_contains?:   string
  limit?:             number
}
```

The archive of thumbs-down events with full provenance. Per-deployment data — never crosses deployment boundaries.

**Errors:** `CommonError`.

---

### `golden_set_results`

```typescript
function golden_set_results(
  set_id:     GoldenSetId,
  snapshot?:  SnapshotId
): Promise<GoldenSetResult | null>
```

Returns the last golden-set run (or a specific snapshot's run, when impl persists per-snapshot results). Returns `null` if the set has never been run.

**Errors:** `CommonError | { kind: "GoldenSetNotFound"; set_id: GoldenSetId }`.

---

### `benchmark_results`

```typescript
function benchmark_results(version?: BenchmarkVersion): Promise<BenchmarkResult | null>
```

Last benchmark run, optionally for a specific version. Returns `null` if benchmarks haven't been run.

**Errors:** `CommonError`.

---

### `run_benchmark`

```typescript
function run_benchmark(version?: BenchmarkVersion): Promise<BenchmarkResult>
```

Run the benchmark suite. Typically invoked by CI on each release.

**Errors:** `CommonError | { kind: "BenchmarkNotConfigured" }`.

**Concurrency:** serialized — at most one benchmark run at a time.

---

### `run_golden_set`

```typescript
function run_golden_set(set_id: GoldenSetId): Promise<GoldenSetResult>
```

Execute a named golden set against the current snapshot.

**Errors:** `CommonError | { kind: "GoldenSetNotFound"; set_id: GoldenSetId }`.

**Concurrency:** serialized per set.

---

### `record_feedback`

```typescript
function record_feedback(call_id: CallId, feedback: FeedbackInput): Promise<void>
```

End-user-supplied feedback (thumbs / structured complaint) is captured against the originating `call_id` (a Synthesis query call). Stored in the failed-query archive if negative.

**Errors:** `CommonError | { kind: "CallNotFound"; call_id: CallId }`.

**Idempotency:** idempotent on `call_id`. Re-recording overwrites prior feedback.

---

### `add_golden_case`

```typescript
function add_golden_case(set_id: GoldenSetId, case_: GoldenCase): Promise<void>
```

Deployer extends a golden set with a new case.

**Errors:** `CommonError | { kind: "GoldenSetNotFound"; set_id: GoldenSetId } | { kind: "DuplicateCase"; case_id: string }`.

**Idempotency:** idempotent on `case_id`. Re-adding the same case is a no-op.

---

### `enable_shadow_eval`

```typescript
function enable_shadow_eval(
  version:      BenchmarkVersion,
  sample_rate?: number    // 0.0–1.0, default 0.05
): Promise<void>
```

Implementer-controlled. Turns on shadow evaluation against the named version with the given sample rate.

**Errors:** `CommonError | { kind: "VersionNotFound"; version: BenchmarkVersion }`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type QualityEvent =
  | { kind: "BenchmarkRun";       payload: { version: BenchmarkVersion; pass: boolean; result_summary: Record<string, number> } }
  | { kind: "GoldenSetRun";       payload: { set_id: GoldenSetId; pass: number; warning: number; fail: number } }
  | { kind: "FeedbackRecorded";   payload: { call_id: CallId; kind: "thumbs_down" | "structured" } }
  | { kind: "ShadowEvalDiff";     payload: { version: BenchmarkVersion; diff_count: number } }
```

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new event kinds, new fields in result types, new `MetricsQuery` dimensions.

## Notes for implementers

- Quality is observation-only. It must never mutate any other container's state.
- The faithfulness evaluator used at runtime by Synthesis is the same component invoked here from `run_golden_set` / `run_benchmark`. Sharing is structural; Quality does not own the runtime invocation path.
- Telemetry aggregation must respect privacy boundaries: cross-deployment aggregates (in a SaaS impl) include only structural signals, never raw content.
