# Cross-Cutting Concerns

These concerns aren't owned by any single container — they shape how all of them are realized in a given deployment. Each section describes the concern conceptually; specific realizations live in `docs/implementations/<impl>/`.

## Multi-tenancy

Deployments may host one tenant or many. The architecture supports both, with a clear partition between **per-tenant state** and **shared (stateless) services**.

| Category | What's per-tenant | What's shared / stateless |
|---|---|---|
| **State** | Raw fragments ([Ingestion](./02-ingestion.md)); atoms + canonical prose + snapshots ([Atoms](./03-atoms.md), incl. its [Projection](./04-projection.md) read facet); trees and their standard-library registrations; blast reports ([Blast Radius](./07-blast-radius.md)); navigation indexes ([Hierarchical Navigation](./06-hierarchical-nav.md)); session / proposal state ([Orchestration](./05-orchestration.md)); failed-query archive + golden sets ([Quality](./10-quality.md)). | None of the above. |
| **Compute** | None inherently. | [Atoms](./03-atoms.md) (extraction, conflict, entailment, [Projection](./04-projection.md) facet), [Blast Radius](./07-blast-radius.md), [Hierarchical Navigation](./06-hierarchical-nav.md). Multi-tenant-aware via the tenant context on each request. |
| **Configuration** | Per-tenant model routing (LLM Gateway), per-tenant golden sets, per-tenant standard-library extensions, per-tenant tree registry. | Implementer-level benchmarks, default routing, default extractors, the standard-library floor. |

The single-tenant case is just the multi-tenant case with a tenant count of one. No architectural divergence — the same containers, the same APIs, just no tenant context to thread.

**Variation points**:

| Variation | Examples |
|---|---|
| Per-tenant state isolation | One repository per tenant (filesystem prefix); one database schema per tenant; one logical namespace within a shared schema. |
| Tenant identification | Subdomain / path prefix / header / session token. |
| Cross-tenant operations | None (strict isolation); implementer-only aggregate telemetry; explicit cross-tenant queries with audit (rare). |

## Tree partitioning within a tenant

Within a single tenant, atoms are further partitioned by `Scope.tree` (e.g., `planning`, `implementation`, `staging`). Trees provide bounded discourses within which same-claim atoms collapse and across which `kind=equivalence` RelationAtoms link parallel claims.

Tree partitioning is **per-tenant**, not cross-tenant. Trees never span tenant boundaries. The tree registry lives inside each tenant's Orchestration state.

## Privacy boundaries

Each external boundary that content crosses is a privacy decision. The architecture identifies these boundaries explicitly so each deployment can apply its own data-handling policy.

| Content | Stays where | Crosses which boundary | Visibility outside the boundary |
|---|---|---|---|
| Raw fragments | Per-tenant [Ingestion](./02-ingestion.md) raw store | The integration source's authentication boundary | The fragment's source still has the original; aala holds a copy. |
| Atoms (canonical) | Per-tenant [Atoms](./03-atoms.md) | The deployer's git host (for snapshot persistence) | Only the deployer (and whoever they grant access). |
| Prose (canonical claim prose + glossary, served via the [Projection](./04-projection.md) facet) | Per-tenant [Atoms](./03-atoms.md) | Same as atoms (committed to the snapshot). | Same as atoms. |
| LLM call content | Per-call | Crosses the LLM provider boundary (whatever the deployer configured in their gateway). | Provider sees prompts and responses; retention depends on provider. |
| Aggregate telemetry | [Quality](./10-quality.md) | Implementer plane (in multi-deployment SaaS) | Latency, conflict-outcome distributions, error rates, tree-bucketed comparisons — no raw content. |
| LLM-as-judge prompts | Per-call | Same boundary as other LLM calls | Provider sees evaluation prompts (which may include atom content for grounding). |
| Failed-query archive | Per-tenant Quality | Tenant boundary only | Internal to the tenant. |

**Variation points**:

| Variation | Examples |
|---|---|
| LLM provider boundary | Cloud provider (top-tier hosted models); local model server (no content leaves the deployment); per-use-case (some use cases use cloud, others local). |
| Telemetry granularity | Aggregate counters only; sampled call records with consent; full prompt+response logs (with redaction). |
| Inter-deployment data sharing | None (each deployment is an island); aggregate signals only; opt-in detailed sharing. |

## Authentication and authorization

Every entry-point call (`Orchestration.ingest`, `Orchestration.query`, `Atoms.update_status`, `Orchestration.register_tree`, etc.) is subject to authentication and authorization. The mechanism varies by deployment shape but the responsibility is consistent.

| Layer | Concern |
|---|---|
| Authentication | Who is calling? End-user via OAuth / SSO / token; agent on behalf of an end-user; CI job with service credentials; implementer admin. |
| Authorization | Is this caller allowed to take this action against this tenant / snapshot / tree / classification scope? Per-tenant role policies (admin / contributor / reader); per-tree policies; per-scope policies. |
| Audit | Every state-changing call is logged with caller identity and rationale, regardless of mechanism. |

This is largely a deployment concern; the architecture doesn't prescribe a specific identity provider or policy engine. What the architecture does prescribe: every container's API surface is callable only through Orchestration's enforced perimeter.

## Observability and tracing

aala produces structured signals at every container boundary:

- **Call records** from [LLM Gateway](./09-llm-gateway.md): use_case_key, latency, tokens, error class.
- **Diff streams** from every container's `changes_since(ref)`: who changed what, when.
- **Telemetry** from [Quality](./10-quality.md): aggregates, golden-set runs, tree-bucketed metrics.
- **Lifecycle events** from [Orchestration](./05-orchestration.md): snapshot transitions, tree lifecycle, ingest lifecycles.
- **Conflict outcomes** from [Atoms](./03-atoms.md): the full conflict pipeline emits structured outcomes that downstream telemetry can aggregate.

**OpenTelemetry emission is mandated**, not optional — it is the observability baseline that lets independently built, composed containers produce one coherent trace (see [10 — Error Model § Mandated observability](../../spec/10-error-model.md#mandated-observability)). Every container MUST emit its signals via OTel using the standard field names, and the operation's `correlation_id` MUST surface as the `trace_id` that threads error returns, logs, and spans together. What remains deployment-specific is *where* signals are persisted and *how long*, not *whether* they are emitted.

The `correlation_id` is a first-class contract element — `trace_id` is returned to callers in errors and written to logs — while OTel **spans** remain pure observability artifacts that correlate *to* that `trace_id`. This guarantees an external system can stitch a complete request trace from agent call to consumer catch-up.

**Variation points**:

| Variation | Examples |
|---|---|
| Span persistence backend | Stdout / structured log file; OpenTelemetry collector pipeline; managed observability platform. (Emission itself is mandatory; the sink is the choice.) |
| Retention | Per-tenant configurable; implementer aggregates only (multi-deployment). |
| Sampling | Always-on; head/tail sampling under load. (Error-tier events SHOULD NOT be sampled out.) |
