# L3 — LLM Gateway Components

For the container framing, see [`L2/09-llm-gateway.md`](../L2/09-llm-gateway.md). LLM Gateway is the one place that knows how aala talks to a model — so no other container does.

## Component diagram

![L3 — LLM Gateway](../diagrams/llmGatewayInternals.png)

## Component reference

| Component | Responsibility | Internal state | Emits / consumes |
|---|---|---|---|
| **Use-Case Key Registry** | The named list of use-case keys aala publishes (`aala.extraction`, `aala.conflict_judge`, `aala.classification_suggest`, `aala.predicate_suggest`, `aala.projection_narrative`, `aala.label_synthesis`, `aala.blast_implicit`, `aala.faithfulness_judge`, `aala.relevance_filter`, `aala.embedding_default`, plus deployment-extensions). Each key has an intent description and a `produces` shape (freeform / structured / embedding). | The registry itself. | Read by `registered_use_cases()`. |
| **Gateway Adapter** | Single concrete adapter for the gateway tool the implementer chose (LiteLLM / OpenRouter / Portkey / custom). Translates aala's `chat(use_case_key, …)` into the tool's native API. | Adapter config (gateway tool endpoint, auth). | In: aala calls. Out: tool-specific requests. |
| **Routing Table** | Per-use-case mapping to primary model + fallback chain + timeout + retry budget. Deployer-configured. | The routing table. | Read by Gateway Adapter and by `routing()` API. |
| **Retry / Timeout Layer** | Applies the per-use-case retry budget and timeout. Uses the gateway tool's native retry where available. | Per-call retry state. | Wraps Gateway Adapter calls. |
| **Fallback Chain** | When configured, on primary-model failure (timeout, rate limit, provider error), attempts the fallback chain in order. Sets `fell_back = true` on success. | None. | Wraps Retry layer. |
| **Schema Validation Layer** | When a caller supplies a JSON Schema, validates the response. On failure, retries with corrective prompts up to the configured budget. After exhausting retries, raises a `generating` / `precondition` Activity Error. | None. | Reads caller schema; validates structured outputs. |
| **Cache Layer** | Opt-in per-call caching keyed by `cache_key`. Returns cached responses on hit; issues fresh model calls on miss. Critical for the projection facet's narrative determinism. | Cache backend (impl-specific). | Read/written per call when `cache_key` supplied. |
| **Observability Emitter** | Emits structured per-call records: use_case_key, model_used, latency_ms, tokens, error_class, retries, fell_back. Destination is deployer-configured. | None. | Outbound to deployer's observability sink. |

## Internal flow — chat call

```mermaid
sequenceDiagram
    participant Caller as Calling Container
    participant Reg as Use-Case Key Registry
    participant Route as Routing Table
    participant Cache as Cache Layer
    participant Schema as Schema Validation
    participant FB as Fallback Chain
    participant Retry as Retry / Timeout
    participant Adapter as Gateway Adapter
    participant Tool as Underlying Gateway Tool
    participant Obs as Observability Emitter

    Caller->>Reg: chat(use_case_key, messages, schema?, options)
    Reg->>Route: lookup primary model + fallback + budgets
    opt options.cache_key present
        Reg->>Cache: lookup
        alt cache hit
            Cache-->>Caller: cached ChatResponse
        else cache miss
            Reg->>FB: invoke chain
        end
    else no cache_key
        Reg->>FB: invoke chain (skip cache)
    end
    FB->>Retry: try primary model
    Retry->>Adapter: send request
    Adapter->>Tool: native API call
    Tool-->>Adapter: response
    Adapter-->>Retry: response
    alt success
        Retry->>Schema: validate (if schema supplied)
        alt schema ok
            Schema-->>FB: validated response
        else schema fail
            Schema->>Retry: retry with corrective prompt
        end
    else timeout / rate limit / provider error
        Retry->>FB: failure
        FB->>FB: try next in fallback chain (set fell_back=true)
    end
    FB->>Obs: emit call record
    opt cache_key supplied
        FB->>Cache: store response
    end
    FB-->>Caller: ChatResponse
```

## Variation points

| Variation | Owned by | Examples |
|---|---|---|
| Concrete gateway tool | Implementer | LiteLLM, OpenRouter, Portkey, custom thin adapter on a provider SDK. |
| Models per use case | Deployer | Configured in the gateway tool. |
| Retry / fallback policy | Implementer (with sensible defaults; may expose knobs) | "extraction: top-tier, on fail retry once at mid-tier." |
| Observability sink | Deployer (where) / Implementer (what to emit) | Stdout / structured logs / metrics endpoint / Quality telemetry. |
| Embedding model | Deployer (per-use-case) | Higher-dim for conflict NN; smaller-dim for fast lookups. |
| Cache backend | Deployer / Implementer | None / in-process / persistent KV. |
| Schema validation strategy | Implementer | Direct (native structured output); retry with corrective prompt; reject on first failure. |
