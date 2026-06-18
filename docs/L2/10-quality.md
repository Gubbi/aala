# Quality — Measurement and Evaluation

**Status: optional, cross-cutting measurement.** A deployment can run without Quality; it simply has no built-in way to measure whether retrieval, conflict classification, faithfulness, or projections are getting better or worse over time.

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
| Benchmark corpus | Synthetic but realistic test inputs maintained by the implementer. Cross-tree scenarios, structured conflict cases (planted disjointness violations, generalization cycles, equivalence ambiguity), multi-hop questions. Versioned with aala releases. |
| Golden-set runner | A deployer authors `{question, expected_sections, expected_answer_traits}`; the runner executes the retrieval over aala's read surfaces (structured [Atoms](./03-atoms.md) reads + the [Projection](./04-projection.md) facet) and scores the result. |
| Retrieval evaluator | Precision and recall over retrieved atoms vs. expected atoms. Deterministic; doesn't depend on prose. |
| Faithfulness evaluator | Checks that every claim asserted in a candidate response appears in the retrieved atom set (or is trivially derivable). Catches confabulation. |
| LLM-as-judge harness | Rubric-based scoring of qualitative dimensions (clarity, completeness, audience fit) via an LLM call. Configurable per dimension. |
| Conflict-pipeline planted-case runner | Injects known structural cases (composition single-parent violations, generalization cycles, disjointness ↔ equivalence contradictions, etc.) and verifies the Conflict pipeline detects them at the right mode (auto-resolve / soft-prompt / hard-block). |
| Shadow eval | When the implementer ships a new version, runs the new version alongside production on a sample of real traffic; logs diffs for sample review. |
| Telemetry aggregator | Aggregate-only signals across deployments (latency distributions, conflict-outcome distributions, error rates, tree-bucketed comparisons). Privacy-respecting — no raw tenant content leaves the deployment. |
| Failed-query archive | Per-deployment store of queries with negative feedback (thumbs-down, structured complaints) + their full provenance (atoms, derived atoms, navigation paths, generated response). Surface for deployer review. |

Quality writes its outputs (eval results, telemetry records, archived failures) either into the snapshot (so a snapshot carries its own quality history) or into Quality's own store (so the running archive spans snapshots). The mix is a variation point.

## API surface (conceptual)

**Read:**
- `metrics(query?) → stats` — aggregate telemetry. Filterable by container, by use case, by time window, by tree.
- `failed_queries(filter?) → archive_entries` — captured failures with provenance.
- `golden_set_results(set_id, snapshot?) → results`
- `benchmark_results(version?) → results`
- `changes_since(ref)` — diff API.

**Write:**
- `run_benchmark(version?) → results`
- `run_golden_set(set_id) → results`
- `record_feedback(call_id, feedback) → ()`
- `add_golden_case(set_id, case) → case_id`
- `enable_shadow_eval(version, sample_rate?) → ()`

## Dependencies

- **[LLM Gateway](./09-llm-gateway.md)** — for LLM-as-judge calls.
- **Every container being measured** — Quality uses their read APIs to evaluate them. It does not write through them. Specifically: [Atoms](./03-atoms.md) (retrieval evaluation, conflict detection tests, faithfulness + qualitative scoring over structured reads and the [Projection](./04-projection.md) facet), [Hierarchical Navigation](./06-hierarchical-nav.md) (navigation-path correctness), [Blast Radius](./07-blast-radius.md) (impact-detection accuracy).
- No dependencies that gate other containers (Quality is observation-only by design).

## Components (preview of L3)

- **Benchmark Corpus** — implementer-maintained synthetic test ground.
- **Benchmark Runner** — runs the corpus on each release, gates deploys on regressions.
- **Golden-Set Runner** — deployer-facing eval surface; executes golden sets and produces scored results.
- **Retrieval Evaluator** — precision/recall against expected atom sets.
- **Faithfulness Evaluator** — claim-grounding check on generated responses.
- **LLM-as-Judge Harness** — rubric-based qualitative scoring.
- **Conflict-Pipeline Test Runner** — injects, runs, verifies structural conflict cases.
- **Shadow Eval Orchestrator** — mirrors traffic to a new version; logs diffs.
- **Telemetry Aggregator** — emits structured aggregates from per-call records.
- **Failed-Query Archive** — captures + indexes thumbs-down events with full provenance.
- **Change Log** — maintains the ordered, append-only event log for this container.

Full L3 detail lives in [`docs/L3/10-quality.md`](../L3/10-quality.md).

## Variation points

| Variation | Owned by | Examples |
|---|---|---|
| Benchmark scope | Implementer | None (skip); basic (smoke tests); full (planted structural conflicts + multi-hop questions + audience-fit cases). |
| Golden-set authoring surface | Implementer | Config files; CLI; web UI in deployer console. |
| Telemetry granularity | Implementer | Counters only (cheap); structured per-call records; full prompt+response logging (with consent + redaction). |
| Tree-bucketed metrics | Implementer | Off (aggregate only); on (compare extraction quality across `planning` vs `implementation`, etc.). |
| Shadow eval | Implementer | Off; sample-rate-based; deterministic on specific call signatures. |
| Judge tier | Implementer / deployer | Cheap-tier judge (fast, less reliable); top-tier judge (accurate, expensive); ensemble. |
| Eval-result persistence | Implementer | In the snapshot alongside atoms; in Quality's own store; both. |

## What Quality does NOT do

- Block or gate other containers — Quality produces signal; CI policy or human review acts on it.
- Mutate atoms, projections, or any canonical state.
- Decide what "good" means in the abstract — rubrics, golden sets, and benchmark expectations are configured. Quality measures against the configured standard.
- Surface evaluations to end-users in real time — Quality is a measurement plane, not a runtime gate. (Composing answers and any runtime checks an external agent performs over aala's read surfaces are the agent's concern, even if the agent reuses the same evaluator components Quality uses.)
