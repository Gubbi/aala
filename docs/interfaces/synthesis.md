# Synthesis — Interface Contract

**Version:** `aala-v1`
**Optionality:** optional
**Container:** [`L2/08-synthesis.md`](../L2/08-synthesis.md) · [`L3/08-synthesis.md`](../L3/08-synthesis.md)

Synthesis composes available capabilities into a single response with provenance: Q&A, ADR drafts, sequence diagrams, walkthroughs, comparisons. It gracefully degrades based on which optional capabilities are wired.

## Types

```typescript
type GeneratorId = string  // e.g., "qa", "adr", "sequence_diagram", "walkthrough", "comparison"

interface GeneratorMeta {
  id:              GeneratorId
  display_name:    string
  description:     string
  floor:           CapabilityRequirement[]   // capabilities required to operate at all
  enhancements:    CapabilityRequirement[]   // capabilities used if present
  operable:        boolean                    // computed: true if all floor capabilities are wired
}

interface CapabilityRequirement {
  capability:      string   // "atoms" | "llm_gateway" | "hierarchical_nav" | "blast_radius" | "projection" | "external_sources"
  use_case_keys?:  UseCaseKey[]
}

interface QueryResponse {
  generator_used:     GeneratorId
  capabilities_used:  string[]
  content:            string         // markdown or generator-specific format
  provenance: {
    atoms:           AtomId[]
    navigation:      string[]
    sources:         string[]
  }
  confidence:         number          // 0.0–1.0
  faithfulness:       "ok" | "warning" | "failed" | "skipped"
  warnings:           string[]        // e.g., "Hierarchical Nav not wired; used brute-force traversal"
}

interface QueryHints {
  generator?:    GeneratorId
  audience?:     "pm" | "designer" | "engineer" | "exec"
  scope_hint?:   ScopePath
  faithfulness_mode?: "off" | "warn" | "block"
}
```

## Methods

### `generators`

```typescript
function generators(): Promise<GeneratorMeta[]>
```

List available generators with their declared floor + enhancements + operability in the current deployment.

**Errors:** `CommonError`.

---

### `query`

```typescript
function query(intent: string, hints?: QueryHints): Promise<QueryResponse>
```

Classify intent, pick a generator, compose capabilities, synthesize a response with provenance.

**Errors:**
```typescript
type QueryError =
  | CommonError
  | { kind: "NoOperableGenerator"; intent: string; reason: string }
  | { kind: "FloorNotMet"; generator: GeneratorId; missing: string[] }
  | { kind: "FaithfulnessFailed"; details: string }    // only when faithfulness_mode = "block"
```

**Idempotency:** not idempotent. Same intent may produce different responses across calls if the underlying atoms changed or non-deterministic LLM calls were involved.

**Concurrency:** concurrent-safe; queries do not mutate any container state.

---

### `changes_since`

Synthesis is largely stateless. Implementations that maintain a response cache or query-history log may emit cache eviction / query-recorded events. Stateless impls return empty pages.

```typescript
type SynthesisEvent =
  | { kind: "ResponseCached";  payload: { intent_hash: string; generator: GeneratorId } }
  | { kind: "CacheEvicted";    payload: { intent_hash: string; reason: string } }
```

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new generators (each impl-declared), new `audience` values, new `faithfulness_mode` values.

## Notes for implementers

- Generators are plug-points: a generator declares its `floor` (must be present) and `enhancements` (used if present) and `query()` checks operability before dispatching.
- Synthesis never writes through to canonical state. Generated output that should become canonical is the caller's responsibility to feed back via `Atoms.propose(...)`.
- Faithfulness checking uses `aala.faithfulness_judge` and is configurable in modes off / warn / block.
