# Ingestion — Interface Contract

**Version:** `aala-v1`
**Optionality:** non-optional
**Container:** [`L2/02-ingestion.md`](../L2/02-ingestion.md) · [`L3/02-ingestion.md`](../L3/02-ingestion.md)

Ingestion is the entry boundary. It accepts normalized fragments, optionally filters for relevance, and persists every accepted fragment durably for provenance.

## Methods

### `accept`

```typescript
function accept(fragment: NormalizedFragment): Promise<AcceptResult>

interface AcceptResult {
  fragment_id:  FragmentId
  accepted:     boolean             // false → dropped by Relevance Filter
  reject_reason?: string            // present when accepted is false
}
```

Runs the relevance filter, persists the fragment to Raw Store if accepted, returns its stable identifier.

**Errors:**
```typescript
type AcceptError =
  | CommonError
  | { kind: "InvalidFragment"; field: string; reason: string }
```

**Idempotency:** idempotent on `fragment.message_id`. Re-accepting the same fragment is a no-op; the original `AcceptResult` may be returned cached.

**Concurrency:** concurrent-safe; per-fragment writes are serialized.

---

### `get`

```typescript
function get(fragment_id: FragmentId): Promise<NormalizedFragment | null>
```

Fetch a fragment with its full metadata. Returns `null` if not present.

**Errors:** `CommonError`.

---

### `list`

```typescript
function list(filter?: FragmentFilter): Promise<NormalizedFragmentMetadata[]>

interface FragmentFilter {
  source_uri_prefix?:  string
  sender?:             string
  content_kind?:       ContentKind
  since?:              string   // ISO 8601
  until?:              string
  limit?:              number
}

interface NormalizedFragmentMetadata {
  fragment_id:   FragmentId
  message_id:    string
  sender:        string
  timestamp:     string
  content_kind:  ContentKind
  source_uri:    string
}
```

Iterate fragment metadata. Payloads are not returned by `list`; fetch them with `get`.

**Errors:** `CommonError`.

---

### `changes_since`

Shared signature from [`00-shared-types.md`](./00-shared-types.md#change-stream-events). Event variants:

```typescript
type IngestionEvent =
  | { kind: "FragmentAccepted"; payload: { fragment_id: FragmentId; metadata: NormalizedFragmentMetadata } }
  | { kind: "FragmentRejected"; payload: { message_id: string; reason: string } }
```

Rarely consumed in steady state; useful for telemetry and re-runs.

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new optional fields in `NormalizedFragment.extras` (free-form by design), new `ContentKind` values, new event kinds.
- Breaking: removing fields, changing idempotency semantics.

## Notes for implementers

- Source adapters and the relevance filter are implementation choices and are not part of this contract. The contract is "given a `NormalizedFragment`, decide accept/reject and persist accepted ones."
- The relevance filter may call out to LLM Gateway with `aala.relevance_filter`; this is observable through LLM Gateway's `usage()` API but not through Ingestion's surface.
