# 09 — LLM Gateway

This chapter binds the LLM Gateway contract: the use-case keys, schema validation, observability, and routing.

## Standard registered use-case keys

A conformant implementation realizing LLM Gateway MUST register at least the standard use-case keys enumerated in [Appendix A § Use-case keys](./appendix-a-registries.md#use-case-keys). Each key has a declared intent and produces a declared shape.

Implementations MAY register additional keys for deployment-specific features. The standard set MUST be present and exposed via `LLMGateway.registered_use_cases()`.

## Routing

`LLMGateway.routing()` MUST return the current mapping from each registered use-case key to:

- A primary model identifier (deployer-supplied).
- An optional fallback chain (ordered alternative model identifiers).
- A timeout (milliseconds).
- A retry budget.

The deployer is the source of model selections; the implementer chooses the underlying gateway tool (LiteLLM, OpenRouter, Portkey, direct provider SDK, custom). The implementer MUST NOT hard-code model selections beyond reasonable defaults that the deployer can override.

## Method semantics

### `chat`

The Gateway MUST:

- Receive the use-case key and consult `routing()` to select the model.
- Issue the chat call to the configured underlying provider, using the gateway tool's structured-output mechanism if a `schema` is supplied.
- If `schema` is supplied, validate the response against it. On validation failure after configured retries, raise a `generating` / `malformed_output` Activity Error.
- Apply the per-use-case timeout. On timeout, raise a `generating` / `timeout` Activity Error (`retry_after_ms` when known), potentially after attempting the fallback chain.
- Emit a structured observability record (see below).

The returned `ChatResponse.content` MUST be schema-conforming when `schema` is supplied. The Gateway MUST NOT return raw text masquerading as structured output.

### `chat_streaming`

The streaming variant MUST emit chunks in order. The final chunk MUST carry `finished = true` and a `final` field populated with the same metadata as a non-streaming `ChatResponse`.

Schema validation in streaming mode happens at the final chunk. If validation fails, the stream MUST terminate with an error chunk.

### `embed`

Returns one vector per input string. The use-case key selects the embedding model — typically `aala.embedding_default` but implementations MAY register additional embedding keys.

The dimensionality is determined by the model. Callers MUST tolerate the model-specific dimension; the contract does not require a fixed value.

### `usage`

Aggregates over all calls made by all callers. Returns latency, token counts, error counts, and fallback frequency by use-case and by model.

## Schema validation

When a caller supplies a `schema` — a [JSON Schema](https://json-schema.org/) document, interpreted under the **draft 2020-12** dialect unless the document's own `$schema` declares another — the Gateway MUST:

1. Pass the schema to the underlying tool when supported (most modern providers support structured-output mode).
2. After receiving the response, validate it against the schema.
3. On validation failure, retry up to the use-case's retry budget. The retry MAY include a corrective prompt asking the model to fix the previous attempt (implementation choice).
4. After exhausting retries, raise a `generating` / `malformed_output` Activity Error, with the last raw content carried on the Primitive `cause` for diagnostics.

The Gateway MUST NOT return content that fails schema validation. Callers can assume schema-conforming responses or an error.

## Fallback chain

When configured, on failure of the primary model (timeout, rate limit, provider error), the Gateway MUST attempt the fallback chain in order. Each fallback attempt is subject to its own timeout. The response, if any fallback succeeds, MUST carry `fell_back = true` so the caller can attach lower confidence or surface a warning.

## Caching

Caching is opt-in per call via `ChatOptions.cache_key`. When `cache_key` is supplied:

- A cache hit MUST return the cached response unchanged.
- A cache miss MUST issue the model call and cache the response under the key for the implementation's TTL.

When `cache_key` is absent, the Gateway MUST NOT consult or update any cache. This is critical for the projection facet's narrative determinism — the cache key encodes `(atom_set + prompt_version + model_version)`, and the absence of `cache_key` indicates the caller wants a fresh call.

## Observability

For every call, the Gateway MUST emit a structured observability record containing at minimum:

- `use_case_key`
- `model_used` (the resolved model id)
- `latency_ms`
- `tokens` (input + output)
- `error_class` (when present)
- `retries`
- `fell_back` (boolean)
- `context_atoms` / `context_fragment` — referenced identifiers only (the call's working atom set; the input fragment id for extraction calls), when the use case has them. Never raw content. These ids let Quality's feedback archive join provenance to the call record ([`interfaces/quality.md § feedback_archive`](../interfaces/quality.md#feedback_archive)).

These records feed Quality telemetry (when wired) and ops dashboards. The destination is deployer-configured.

## No content persistence

The Gateway MUST NOT persist prompt or response content beyond the cache (when opted in via `cache_key`) and the structured observability records. In particular: it MUST NOT log raw content by default. Implementations MAY offer prompt+response logging as a deployer-configured option with consent and redaction.

## Privacy boundary

LLM Gateway is the only container that legitimately sends content to external LLM providers. Every other container reaches the LLM only through the Gateway. The deployer-configured routing is the single point where content's privacy boundary is set (which models the deployer trusts to receive content).
