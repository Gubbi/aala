# LLM Gateway — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** cross-cutting infrastructure (required by any deployment running LLM calls)

LLM Gateway is the unified abstraction for language-model access. Callers specify a use-case key; the implementer-chosen gateway tool routes it to a deployer-configured model.

## Types

```typescript
type Role = "system" | "user" | "assistant" | "tool"

interface Message {
  role:        Role
  content:     string
  name?:       string
  tool_calls?: ToolCall[]    // when role is "assistant"
  tool_call_id?: string      // when role is "tool"
}

interface ToolCall {
  id:        string
  name:      string
  arguments: Record<string, unknown>
}

type JsonSchema = Record<string, unknown>  // a JSON Schema document, draft 2020-12 dialect (spec/09 § Schema validation)

interface ChatOptions {
  temperature?:        number
  max_tokens?:         number
  stop?:               string[]
  timeout_ms?:         number              // override default
  retry_budget?:       number              // override default
  allow_fallback?:     boolean             // default true
  cache_key?:          string              // opt-in caching; absent means no cache
}

interface ChatResponse<T = unknown> {
  content:         T                       // string when no schema; parsed object when schema provided
  raw_content?:    string                  // raw model output (when caller wants both)
  model_used:      string                  // resolved model id from routing
  tokens:          { input: number; output: number }
  latency_ms:      number
  fell_back:       boolean                 // true if the call hit the fallback chain
  retries:         number
}

interface EmbedResponse {
  vectors:         number[][]              // one per input
  model_used:      string
  dimensions:      number
  tokens:          { input: number }
}

interface UsageStats {
  by_use_case:     Record<UseCaseKey, UseCaseStats>
  by_model:        Record<string, ModelStats>
  total: {
    calls:         number
    tokens:        { input: number; output: number }
    errors:        number
  }
}

interface UseCaseStats {
  calls:           number
  p50_ms:          number
  p99_ms:          number
  tokens:          { input: number; output: number }
  errors:          number
  fell_back:       number
}

interface ModelStats extends UseCaseStats {
  model_id:        string
}

interface UseCaseInfo {
  key:             UseCaseKey
  intent:          string                  // what aala calls this use case for
  produces:        "freeform" | "structured" | "embedding"
}
```

## Methods

### `chat`

```typescript
function chat<T = string>(
  use_case_key: UseCaseKey,
  messages:     Message[],
  schema?:      JsonSchema,
  options?:     ChatOptions
): Promise<ChatResponse<T>>
```

Issue a chat call routed by use-case key. With `schema`, the gateway validates the response and returns a parsed object; without it, returns raw text. Validation failures (after configured retries) raise `generating` / `malformed_output`, whose Primitive `cause` carries the last raw content.

**Errors:** `generating` — `unknown_use_case` (the `use_case_key` is not registered), `malformed_output` (model output fails the response schema after the gateway's internal retries), `throttled` (provider rate limit; `retry_after_ms` SHOULD be set), `timeout` (call deadline exceeded), `provider_failure` (the provider returned an error).

**Access:** `maintain` — the gateway's contract callers are aala containers executing within an already-admitted operation, which is not re-checked ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)); where a deployment exposes this method at the perimeter, it is an operator surface.

**Idempotency:** not idempotent in general (model outputs are non-deterministic). Idempotent only when `options.cache_key` is supplied and a cache hit is taken — the cached response is returned unchanged. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** concurrent-safe.

---

### `chat_streaming`

```typescript
function chat_streaming<T = string>(
  use_case_key: UseCaseKey,
  messages:     Message[],
  schema?:      JsonSchema,
  options?:     ChatOptions
): Stream<ChatChunk<T>>

interface ChatChunk<T> {
  delta:           Partial<T> | string     // structured partial when schema provided, else string delta
  index:           number                  // monotonic
  finished:        boolean                 // true on the last chunk
  finish_reason?:  "stop" | "length" | "tool_call" | "error"
  final?:          ChatResponse<T>         // present in the final chunk
}
```

Streaming variant of `chat`. The final chunk carries the same metadata as a non-streaming `ChatResponse`.

**Errors:** same as `chat`, surfaced as a stream-terminating error event.

**Access:** `maintain` (as `chat`).

---

### `embed`

```typescript
function embed(
  use_case_key: UseCaseKey,
  text:         string | string[]
): Promise<EmbedResponse>
```

Returns one vector per input string. The use-case key selects the embedding model.

**Errors:** `generating` — `unknown_use_case` (the `use_case_key` is not registered), `throttled`, `timeout`, `provider_failure`.

**Access:** `maintain` (as `chat`).

**Concurrency:** concurrent-safe.

---

### `usage`

```typescript
function usage(timeframe?: { since: string; until?: string }): Promise<UsageStats>
```

Aggregate call statistics for ops + Quality consumption.

**Errors:** `querying` — `invalid_query` (malformed timeframe).

**Access:** `query`.

---

### `routing`

```typescript
function routing(): Promise<RoutingTable>

interface RoutingTable {
  entries:  Record<UseCaseKey, RoutingEntry>
}

interface RoutingEntry {
  use_case_key:    UseCaseKey
  primary_model:   string
  fallback_chain:  string[]
  timeout_ms:      number
  retry_budget:    number
}
```

Returns the current use-case → model mapping. Useful for debugging "why did this use case use that model?"

**Errors:** `querying` — universal end-states only.

**Access:** `query`.

---

### `registered_use_cases`

```typescript
function registered_use_cases(): Promise<UseCaseInfo[]>
```

The published list of use-case keys aala expects to call. Consumed by deployers configuring their gateway tool — they need to know which keys to wire up.

**Errors:** `querying` — universal end-states only.

**Access:** `query`.

---

## Standard registered use-case keys

The standard keys — part of the `aala-v0.1` published contract — are enumerated in [Appendix A § Use-case keys](../spec/appendix-a-registries.md#use-case-keys), with semantics in [09 — LLM Gateway](../spec/09-llm-gateway.md#standard-registered-use-case-keys). Deployers MUST map a model to each standard key in their gateway.

Implementations MAY register additional keys for deployment-specific features. Standard keys MUST be present in `registered_use_cases()`.

---

## Version notes

- **`aala-v0.1`** — initial contract with the standard keys ([Appendix A § Use-case keys](../spec/appendix-a-registries.md#use-case-keys)).
- Backward-compatible: new standard keys (each clearly documented), new optional fields in `ChatOptions` / `ChatResponse` / `UseCaseInfo`.
- Breaking: removing standard keys; changing what an existing key is expected to produce.

## Notes for implementers

- The Gateway is schema-transparent: callers provide the schema, the gateway forwards to the underlying tool (using the tool's structured-output mechanism when available) and validates the response.
- Caching is opt-in per call via `options.cache_key`. The projection facet's narrative rendering relies on this for determinism.
- Observability emission (call records to Quality) is part of the contract — implementations MUST emit per-call structured records when any consumer subscribes, each keyed by a freshly minted `CallId` ([Appendix A § Identifier minting](../spec/appendix-a-registries.md#identifier-minting)).
- No content persistence beyond the cache and structured observability records. Raw content MUST NOT be logged by default. Optional deployer-configured logging with consent and redaction is permitted.
