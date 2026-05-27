# LLM Gateway — Interface Contract

**Version:** `aala-v1`
**Optionality:** cross-cutting infrastructure (required by any deployment running LLM calls)
**Container:** [`L2/09-llm-gateway.md`](../L2/09-llm-gateway.md) · [`L3/09-llm-gateway.md`](../L3/09-llm-gateway.md)

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

type JsonSchema = Record<string, unknown>  // standard JSON Schema document

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

Issue a chat call routed by use-case key. With `schema`, the gateway validates the response and returns a parsed object; without it, returns raw text. Validation failures (after configured retries) raise `SchemaViolation`.

**Errors:**
```typescript
type ChatError =
  | CommonError
  | { kind: "UnknownUseCase"; use_case_key: UseCaseKey }
  | { kind: "SchemaViolation"; reason: string; raw_content: string }
  | { kind: "Timeout"; elapsed_ms: number }
  | { kind: "RateLimited"; retry_after_ms?: number }
  | { kind: "ProviderError"; provider: string; reason: string }
```

**Idempotency:** not idempotent in general (LLMs are non-deterministic). Idempotent when `options.cache_key` is supplied and a cache hit is taken; the gateway returns the cached response.

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

---

### `embed`

```typescript
function embed(
  use_case_key: UseCaseKey,
  text:         string | string[]
): Promise<EmbedResponse>
```

Returns one vector per input string. The use-case key selects the embedding model.

**Errors:**
```typescript
type EmbedError =
  | CommonError
  | { kind: "UnknownUseCase"; use_case_key: UseCaseKey }
  | { kind: "ProviderError"; provider: string; reason: string }
```

**Concurrency:** concurrent-safe.

---

### `usage`

```typescript
function usage(timeframe?: { since: string; until?: string }): Promise<UsageStats>
```

Aggregate call statistics for ops + Quality consumption.

**Errors:** `CommonError`.

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

**Errors:** `CommonError`.

---

### `registered_use_cases`

```typescript
function registered_use_cases(): Promise<UseCaseInfo[]>
```

The published list of use-case keys aala expects to call. Consumed by deployers configuring their gateway tool — they need to know which keys to wire up.

**Errors:** `CommonError`.

---

## Standard registered use-case keys

These are part of the `aala-v1` published contract; deployers must map a model to each one in their gateway:

| Key | Intent | Produces |
|---|---|---|
| `aala.extraction` | Atom extraction from a fragment | structured |
| `aala.conflict_judge` | Pairwise atom conflict classification | structured |
| `aala.blast_implicit` | Impact-analysis implicit pass | structured |
| `aala.projection_narrative` | Render connective prose anchored to previous projection | freeform |
| `aala.label_synthesis` | Generate semantic node labels for tree axes | freeform |
| `aala.synthesis` | Multi-purpose synthesis (intent classification, response composition, diagram generation) | freeform or structured |
| `aala.faithfulness_judge` | Verify response claims grounded in retrieved atoms | structured |
| `aala.relevance_filter` | Pre-extraction relevance classification | structured (binary) |
| `aala.embedding_default` | Default embedding model | embedding |

Implementations may register additional keys for deployment-specific features. Standard keys must be present in `registered_use_cases()`.

---

## Version notes

- **`aala-v1`** — initial contract with the nine standard keys above.
- Patch-compatible: new standard keys (each clearly documented), new optional fields in `ChatOptions` / `ChatResponse` / `UseCaseInfo`.
- Breaking: removing standard keys; changing what an existing key is expected to produce.

## Notes for implementers

- The Gateway is schema-transparent: callers provide the schema, the gateway forwards to the underlying tool (using the tool's structured-output mechanism when available) and validates the response.
- Caching is opt-in per call via `options.cache_key`. Projection's narrative rendering relies on this for determinism.
- Observability emission (call records to Quality) is part of the contract — implementations must emit per-call structured records when any consumer subscribes.
