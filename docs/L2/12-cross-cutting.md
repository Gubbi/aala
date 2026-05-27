# Cross-Cutting Concerns

These concerns aren't owned by any single container — they shape how all of them are realized in a given deployment. Each section describes the concern conceptually; specific realizations live in `docs/implementations/<impl>/`.

## Multi-tenancy

Deployments may host one tenant or many. The architecture supports both, with a clear partition between **per-tenant state** and **shared (stateless) services**.

| Category | What's per-tenant | What's shared / stateless |
|---|---|---|
| **State** | Raw fragments ([Ingestion](./02-ingestion.md)); atoms + projections + snapshots ([Atoms](./03-atoms.md), [Projection](./04-projection.md)); blast reports ([Blast Radius](./07-blast-radius.md)); navigation indexes ([Hierarchical Navigation](./06-hierarchical-nav.md)); session / proposal state ([Orchestration](./05-orchestration.md)); failed-query archive + golden sets ([Quality](./10-quality.md)). | None of the above. |
| **Compute** | None inherently. | [Atoms](./03-atoms.md) (extraction, conflict), [Projection](./04-projection.md), [Synthesis](./08-synthesis.md), [Blast Radius](./07-blast-radius.md), [Hierarchical Navigation](./06-hierarchical-nav.md). Multi-tenant-aware via the tenant context on each request. |
| **Configuration** | Per-tenant model routing (LLM Gateway), per-tenant golden sets, per-tenant taxonomy. | Implementer-level benchmarks, default routing, default extractors. |

The single-tenant case is just the multi-tenant case with a tenant count of one. No architectural divergence — the same containers, the same APIs, just no tenant context to thread.

**Variation points**:

| Variation | Examples |
|---|---|
| Per-tenant state isolation | One repository per tenant (filesystem prefix); one database schema per tenant; one logical namespace within a shared schema. |
| Tenant identification | Subdomain / path prefix / header / session token. |
| Cross-tenant operations | None (strict isolation); implementer-only aggregate telemetry; explicit cross-tenant queries with audit (rare). |

## Privacy boundaries

Each external boundary that content crosses is a privacy decision. The architecture identifies these boundaries explicitly so each deployment can apply its own data-handling policy.

| Content | Stays where | Crosses which boundary | Visibility outside the boundary |
|---|---|---|---|
| Raw fragments | Per-tenant [Ingestion](./02-ingestion.md) raw store | The integration source's authentication boundary | The fragment's source still has the original; aala holds a copy. |
| Atoms (canonical) | Per-tenant [Atoms](./03-atoms.md) | The deployer's git host (for snapshot persistence) | Only the deployer (and whoever they grant access). |
| Prose projections | Per-tenant [Projection](./04-projection.md) | Same as atoms (committed to the snapshot). | Same as atoms. |
| LLM call content | Per-call | Crosses the LLM provider boundary (whatever the deployer configured in their gateway). | Provider sees prompts and responses; retention depends on provider. |
| Aggregate telemetry | [Quality](./10-quality.md) | Implementer plane (in multi-deployment SaaS) | Latency, classification distributions, error rates — no raw content. |
| LLM-as-judge prompts | Per-call | Same boundary as other LLM calls | Provider sees evaluation prompts (which may include atom content for grounding). |
| Failed-query archive | Per-tenant Quality | Tenant boundary only | Internal to the tenant. |

**Variation points**:

| Variation | Examples |
|---|---|
| LLM provider boundary | Cloud provider (top-tier hosted models); local model server (no content leaves the deployment); per-use-case (some use cases use cloud, others local). |
| Telemetry granularity | Aggregate counters only; sampled call records with consent; full prompt+response logs (with redaction). |
| Inter-deployment data sharing | None (each deployment is an island); aggregate signals only; opt-in detailed sharing. |

## Authentication and authorization

Every entry-point call (`Orchestration.ingest`, `Orchestration.query`, `Atoms.update_status`, etc.) is subject to authentication and authorization. The mechanism varies by deployment shape but the responsibility is consistent.

| Layer | Concern |
|---|---|
| Authentication | Who is calling? End-user via OAuth / SSO / token; agent on behalf of an end-user; CI job with service credentials; implementer admin. |
| Authorization | Is this caller allowed to take this action against this tenant / snapshot / atom scope? Per-tenant role policies (admin / contributor / reader); per-scope policies (e.g., a contributor only writes to certain domain scopes). |
| Audit | Every state-changing call is logged with caller identity and rationale, regardless of mechanism. |

This is largely a deployment concern; the architecture doesn't prescribe a specific identity provider or policy engine. What the architecture does prescribe: every container's API surface is callable only through Orchestration's enforced perimeter.

## Observability and tracing

aala produces structured signals at every container boundary:

- **Call records** from [LLM Gateway](./09-llm-gateway.md): use_case_key, latency, tokens, error class.
- **Diff streams** from every container's `changes_since(ref)`: who changed what, when.
- **Telemetry** from [Quality](./10-quality.md): aggregates and golden-set runs.
- **Lifecycle events** from [Orchestration](./05-orchestration.md): snapshot transitions, ingest lifecycles.

How these are correlated, persisted, and visualized is deployment-specific. The architecture's commitment is that every container emits enough structured signal that an external tracing system can stitch a complete request trace from agent call to consumer catch-up.

**Variation points**:

| Variation | Examples |
|---|---|
| Trace correlation | Per-request trace ID propagated through every container; per-snapshot ref correlation; no tracing (minimal deployments). |
| Persistence | Stdout / structured log file; collector pipeline (e.g., OpenTelemetry collector); managed observability platform. |
| Retention | Per-tenant configurable; implementer aggregates only (multi-deployment). |
