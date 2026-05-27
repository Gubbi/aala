# Quality — Measurement and Evaluation

**Status: optional, cross-cutting measurement.** A deployment can run without Quality; it simply has no built-in way to measure whether retrieval, conflict classification, generated answers, or projections are getting better or worse over time.

## Concern

Quality measures the system's outputs. It runs benchmark suites and golden sets, scores responses for retrieval precision and faithfulness, captures and surfaces failures, aggregates telemetry across calls. The container does not gate or block other containers — it produces signal; humans or CI policies act on it.

In one sentence: **Quality is the one container that knows whether the other containers are doing a good job — and accumulates evidence over time.**

## Layering — who uses Quality

| Role | Uses |
|---|---|
| **Implementer (aala)** | Maintains the benchmark corpus; runs the benchmark suite on every release (CI gate against quality regressions); monitors aggregate telemetry from deployments (privacy-respecting). |
| **Deployer (end-user)** | Authors golden sets specific to their domain (questions + expected retrieved-section IDs + expected answer traits); reviews the failed-query archive; runs domain-specific evaluations. |
| **End-users** | Submit feedback (thumbs / structured) that feeds the failed-query archive. |

## What it owns

| Concern | Notes |
|---|---|
| Benchmark corpus | Synthetic but realistic test inputs maintained by the implementer. A handful of services, ADRs with rationale, planted contradictions, multi-hop questions. Versioned with aala releases. |
| Golden-set runner | A deployer authors `{question, expected_sections, expected_answer_traits}`; the runner executes the question against [Synthesis](./08-synthesis.md) (or directly against retrieval) and scores the result. |
| Retrieval evaluator | Precision and recall over retrieved atoms vs. expected atoms. Deterministic; doesn't depend on prose. |
| Faithfulness evaluator | Checks that every claim asserted in a generated response appears in the retrieved atom set (or is trivially derivable). Catches confabulation. |
| LLM-as-judge harness | Rubric-based scoring of qualitative dimensions (clarity, completeness, audience fit) via an LLM call. Configurable per dimension. |
| Planted-contradiction runner | Injects known contradictions into the system; verifies that Conflict (inside [Atoms](./03-atoms.md)) detects them. End-to-end check on the conflict pipeline. |
| Shadow eval | When the implementer ships a new version, runs the new version alongside production on a sample of real traffic; logs diffs for sample review. Lets the implementer catch regressions before fully rolling out. |
| Telemetry aggregator | Aggregate-only signals across deployments (latency distributions, classification distributions, error rates). Privacy-respecting — no raw tenant content leaves the deployment. |
| Failed-query archive | Per-deployment store of queries with negative feedback (thumbs-down, structured complaints) + their full provenance (atoms, navigation paths, generated response). Surface for deployer review. |

Quality writes its outputs (eval results, telemetry records, archived failures) either into the snapshot (so a snapshot carries its own quality history) or into Quality's own store (so the running archive spans snapshots). The mix is a variation point.

## API surface (conceptual)

**Read:**
- `metrics(timeframe?, by?) → stats` — aggregate telemetry (call counts, latency, error rates, classification distributions). Filterable by container, by use case, by time window.
- `failed_queries(filter?) → archive_entries` — captured failures with provenance.
- `golden_set_results(set_id, snapshot?) → results` — last (or historical) run of a named golden set.
- `benchmark_results(version?) → results` — last (or historical) benchmark run.
- `changes_since(ref)` — diff API.

**Write:**
- `run_benchmark(version?) → results` — run the benchmark suite. Typically invoked by CI.
- `run_golden_set(set_id) → results` — run a named golden set. Typically invoked by a deployer.
- `record_feedback(call_id, feedback) → ()` — end-user-supplied feedback (thumbs, structured complaints).
- `add_golden_case(set_id, case) → case_id` — deployer extends a golden set.
- `enable_shadow_eval(version, sample_rate?) → ()` — implementer-controlled.

## Dependencies

- **[LLM Gateway](./09-llm-gateway.md)** — for LLM-as-judge calls.
- **Every container being measured** — Quality uses their read APIs to evaluate them. It does not write through them. Specifically: [Atoms](./03-atoms.md) (retrieval evaluation, conflict detection tests), [Projection](./04-projection.md) (rendered-output checks), [Synthesis](./08-synthesis.md) (faithfulness, qualitative scoring), [Hierarchical Navigation](./06-hierarchical-nav.md) (navigation-path correctness), [Blast Radius](./07-blast-radius.md) (impact-detection accuracy).
- No dependencies that gate other containers (Quality is observation-only by design).

## Components (preview of L3)

- **Benchmark Corpus** — implementer-maintained synthetic test ground.
- **Benchmark Runner** — runs the corpus on each release, gates deploys on regressions.
- **Golden-Set Runner** — deployer-facing eval surface; executes golden sets and produces scored results.
- **Retrieval Evaluator** — precision/recall against expected atom sets.
- **Faithfulness Evaluator** — claim-grounding check on generated responses.
- **LLM-as-Judge Harness** — rubric-based qualitative scoring.
- **Planted-Contradiction Runner** — injects, runs, verifies.
- **Shadow Eval Orchestrator** — mirrors traffic to a new version; logs diffs.
- **Telemetry Aggregator** — emits structured aggregates from per-call records.
- **Failed-Query Archive** — captures + indexes thumbs-down events with full provenance.
- **Change Log** — maintains the ordered, append-only event log for this container. Receives `BenchmarkRun` / `GoldenSetRun` / `FeedbackRecorded` / `ShadowEvalDiff` notifications from the runners and archive. Serves `changes_since(ref)` for dashboards, CI gates, and external observability tooling.

Full L3 detail lives in `docs/L3/10-quality.md` (TBD).

## Variation points

| Variation | Owned by | Examples |
|---|---|---|
| Benchmark scope | Implementer | None (skip); basic (smoke tests); full (planted contradictions + multi-hop questions + audience-fit cases). |
| Golden-set authoring surface | Implementer | Config files; CLI; web UI in deployer console. |
| Telemetry granularity | Implementer | Counters only (cheap); structured per-call records; full prompt+response logging (with consent + redaction). |
| Shadow eval | Implementer | Off; sample-rate-based; deterministic on specific call signatures. |
| Judge tier | Implementer / deployer | Cheap-tier judge (fast, less reliable); top-tier judge (accurate, expensive); ensemble. |
| Eval-result persistence | Implementer | In the snapshot alongside atoms; in Quality's own store; both. |

## What Quality does NOT do

- Block or gate other containers — Quality produces signal; CI policy or human review acts on it.
- Mutate atoms, projections, or any canonical state.
- Decide what "good" means in the abstract — rubrics, golden sets, and benchmark expectations are configured (by the implementer for benchmarks, by the deployer for golden sets). Quality measures against the configured standard.
- Surface evaluations to end-users in real time — Quality is a measurement plane, not a runtime gate. (A runtime faithfulness check during response synthesis is owned by [Synthesis](./08-synthesis.md), even if it consults the same evaluator components Quality uses.)
