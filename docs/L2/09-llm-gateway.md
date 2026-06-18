# LLM Gateway — Unified Model Access

**Status: cross-cutting infrastructure.** Required by every deployment that runs language-model calls (i.e., every deployment with [Atoms](./03-atoms.md) extraction / classification suggestion / predicate suggestion / conflict judging, [projection](./04-projection.md) narrative rendering inside a contentful container's projection facet, [Hierarchical Navigation](./06-hierarchical-nav.md) label synthesis, [Blast Radius](./07-blast-radius.md) implicit pass, or [Quality](./10-quality.md) LLM-as-judge).

## Concern

LLM Gateway is the single seam between aala and any underlying LLM access. Other containers issue calls through this container — they never speak directly to a model provider.

In one sentence: **LLM Gateway is the one place that knows how aala talks to a model — so no other container does.**

## Layering — who decides what

| Role | Owns |
|---|---|
| **Implementer (aala)** | Picks the concrete gateway tool aala uses (LiteLLM, OpenRouter, Portkey, or a custom adapter). Defines the registered set of use-case keys. Implements retries, fallback chains, observability emission. |
| **Deployer (end-user)** | Configures the chosen gateway with their own models: which model serves each use-case key. Brings their own provider subscriptions / hosted endpoints / local model servers. |
| **Caller (each aala container that issues LLM calls)** | Owns its prompt and its expected response schema. Passes both to the gateway; gateway passes them through. |

The deployer's surface is intentionally narrow: a configuration file mapping `aala.<use_case>` → `<model>` in their LLM gateway tool. They never have to learn aala's internals to choose models.

## What it owns

| Concern | Notes |
|---|---|
| Use-case key registry | The named buckets aala calls: `aala.extraction`, `aala.conflict_judge`, `aala.classification_suggest`, `aala.predicate_suggest`, `aala.projection_narrative`, `aala.label_synthesis`, `aala.blast_implicit`, `aala.faithfulness_judge`, `aala.relevance_filter`, `aala.embedding_default`, plus any deployment-extension keys. The registry is part of aala's published contract. |
| Adapter to the chosen gateway tool | Thin layer that turns aala's `chat(use_case_key, messages, schema?)` into a request the configured gateway tool understands. One adapter per supported gateway tool. |
| Retry / timeout policy | Per-use-case timeouts and retry budgets. Set by the implementer with sensible defaults; using the gateway tool's retry features where available. |
| Fallback chain | When configured, "if top-tier model fails or times out for a use case, retry against a lower-tier model with a flag set on the response." |
| Observability | Structured records per call: use_case_key, latency, tokens, error class, retry count. Emitted to [Quality](./10-quality.md) telemetry and to operational tooling. |

LLM Gateway does **not** own schemas. Each caller passes its own; the gateway is schema-transparent (forwards to the underlying tool's structured-output mechanism when available; otherwise, the caller validates the response itself).

## API surface (conceptual)

- `chat(use_case_key, messages, schema?, options?) → response` — caller's schema is passed through. Response is the model's output (schema-conforming if a schema was provided and the underlying tool supports structured outputs).
- `chat_streaming(use_case_key, messages, schema?, options?) → stream<chunk>` — streaming variant.
- `embed(use_case_key, text | texts[]) → vector | vectors[]` — embeddings; the use-case key selects the embedding model.
- `usage(timeframe?, by?) → stats` — observability surface for ops.
- `routing() → routing_table` — current use-case → model mapping.
- `registered_use_cases() → [use_case_info]` — what aala publishes so deployers know what they need to configure.

Callers never name a model or provider directly — they name a use case.

## Dependencies

- **One configured underlying gateway tool** — variation point; see below.
- No dependencies on other aala containers (the relationship is inverted: many containers depend on the Gateway).

## Components (preview of L3)

- **Use-Case Key Registry** — the named list, with intent descriptions, that the implementer publishes and deployers reference.
- **Gateway Adapter** — single concrete adapter for the gateway tool chosen by the implementer. Translates `chat(use_case_key, …)` into the tool's native API.
- **Retry / Timeout Layer** — applies the per-use-case policy.
- **Fallback Chain** — uses the gateway tool's tier mechanisms where present.
- **Schema Validation Layer** — validates structured responses against caller-supplied schemas; retries with corrective prompts up to the configured budget.
- **Observability Emitter** — structured call records.
- **Cache Layer** — opt-in per-call caching keyed by `cache_key`.

Full L3 detail lives in [`docs/L3/09-llm-gateway.md`](../L3/09-llm-gateway.md).

## Variation points

| Variation | Owned by | Examples |
|---|---|---|
| Concrete gateway tool | Implementer | LiteLLM, OpenRouter, Portkey, a custom thin adapter that talks directly to a provider SDK. |
| Models per use case | Deployer | Configured in the gateway tool. The deployer brings their own subscriptions / endpoints / local model servers. |
| Retry / fallback policy | Implementer (with sensible defaults; may expose knobs) | "extraction: top-tier, on fail retry once at mid-tier with confidence-flag." |
| Observability sink | Deployer (where to send) / Implementer (what to emit) | Stdout / structured logs / metrics endpoint / [Quality](./10-quality.md) telemetry. |
| Embedding model | Deployer (per-use-case) | Higher-dim for conflict-NN; smaller-dim for fast lookups. |
| Caching | Deployer (TTL + backend) / Caller (opt-in per call) | None / in-process / persistent KV. |

## What LLM Gateway does NOT do

- Decide *what* to put in a prompt — that's the calling container's job.
- Hold a schema registry — schemas live with the calling container that defines them.
- Make business judgments — those are caller concerns.
- Pick models — that's the deployer's configuration in their gateway tool.
- Persist call content into the canonical record — calls are ephemeral; their *results* may become atoms, but only via the normal `Atoms.propose` path.
