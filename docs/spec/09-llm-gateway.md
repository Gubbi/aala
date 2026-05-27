# 09 — LLM Gateway

This chapter binds the LLM Gateway contract: the use-case keys, schema validation, observability, and routing.

## Standard registered use-case keys

A conformant implementation realizing LLM Gateway MUST register at least the following keys. Each key has a declared intent and produces a declared shape:

| Key | Intent | Produces |
|---|---|---|
| `aala.extraction` | Extract atoms from a `NormalizedFragment`. | structured (matches the atom schema) |
| `aala.conflict_judge` | Classify a proposed atom against a canonical candidate (one of the 8 outcomes). | structured |
| `aala.blast_implicit` | "Does atom Y still hold under the new premise from decision X?" | structured |
| `aala.projection_narrative` | Render connective prose anchored to the previous projection. | freeform |
| `aala.label_synthesis` | Generate a semantic label + optional description for a navigation tree node. | freeform |
| `aala.synthesis` | Multi-purpose: intent classification, response composition, diagram generation. The schema is per-call (varies by sub-use). | freeform or structured (per call) |
| `aala.faithfulness_judge` | Verify a generated response's claims against a retrieved atom set. | structured |
| `aala.relevance_filter` | Classify a fragment as relevant / not-relevant for downstream extraction. | structured (binary or scored) |
| `aala.embedding_default` | Default text-to-vector embedding model. | embedding |

Implementations MAY register additional keys for deployment-specific features. The standard 9 MUST be present and exposed via `LLMGateway.registered_use_cases()`.

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
- If `schema` is supplied, validate the response against it. On validation failure after configured retries, raise `SchemaViolation`.
- Apply the per-use-case timeout. On timeout, raise `Timeout` (potentially after attempting the fallback chain).
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

When a caller supplies a `schema` (JSON Schema), the Gateway MUST:

1. Pass the schema to the underlying tool when supported (most modern providers support structured-output mode).
2. After receiving the response, validate it against the schema.
3. On validation failure, retry up to the use-case's retry budget. The retry MAY include a corrective prompt asking the model to fix the previous attempt (implementation choice).
4. After exhausting retries, raise `SchemaViolation` with the last raw content for diagnostics.

The Gateway MUST NOT return content that fails schema validation. Callers can assume schema-conforming responses or an error.

## Fallback chain

When configured, on failure of the primary model (timeout, rate limit, provider error), the Gateway MUST attempt the fallback chain in order. Each fallback attempt is subject to its own timeout. The response, if any fallback succeeds, MUST carry `fell_back = true` so the caller can attach lower confidence or surface a warning.

## Caching

Caching is opt-in per call via `ChatOptions.cache_key`. When `cache_key` is supplied:

- A cache hit MUST return the cached response unchanged.
- A cache miss MUST issue the model call and cache the response under the key for the implementation's TTL.

When `cache_key` is absent, the Gateway MUST NOT consult or update any cache. This is critical for Projection's determinism — the cache key encodes `(atom_set + prompt_version + model_version)`, and the absence of `cache_key` indicates the caller wants a fresh call.

## Observability

For every call, the Gateway MUST emit a structured observability record containing at minimum:

- `use_case_key`
- `model_used` (the resolved model id)
- `latency_ms`
- `tokens` (input + output)
- `error_class` (when present)
- `retries`
- `fell_back` (boolean)

These records feed [Quality](./10-error-model.md) telemetry (when wired) and ops dashboards. The destination is deployer-configured.

## No content persistence

The Gateway MUST NOT persist prompt or response content beyond the cache (when opted in via `cache_key`) and the structured observability records. In particular: it MUST NOT log raw content by default. Implementations MAY offer prompt+response logging as a deployer-configured option with consent and redaction.

## Privacy boundary

LLM Gateway is the only container that legitimately sends content to external LLM providers. Every other container reaches the LLM only through the Gateway. The deployer-configured routing is the single point where content's privacy boundary is set (which models the deployer trusts to receive content).
