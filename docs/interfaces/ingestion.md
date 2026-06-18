# Ingestion — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** non-optional

Ingestion is the entry boundary. It accepts normalized fragments, optionally filters for relevance, and persists every accepted fragment durably for provenance.

## Methods

### `accept`

```typescript
function accept(fragment: NormalizedFragment): Promise<AcceptResult>

interface AcceptResult {
  ingest_id:      IngestId            // the ingest:<opaque> correlation id stamped on this operation
  accepted:       boolean             // false → dropped by Relevance Filter
  fragment_id?:   FragmentId          // present iff accepted — a rejected fragment is never persisted and has no FragmentId
  reject_reason?: string              // present iff accepted is false
}
```

Runs the relevance filter and, when the fragment is accepted, persists it to Raw Store under a stable `FragmentId`. Each evaluation mints the `ingest:<opaque>` correlation id that is stamped on every change event the operation causes ([05 — Delta Streams § Correlation](../spec/05-delta-streams.md#correlation-across-streams)); the id is returned as `ingest_id` so the caller — typically [`Orchestration.ingest`](./orchestration.md#ingest) — can thread it downstream and report it in its own summary.

A rejected fragment is **not persisted**: no `FragmentId` exists for it, and it is identified by `fragment.message_id` alone — in the result and in the `FragmentRejected` event. If the relevance filter cannot run (its model call fails after the gateway's retries, or a dependency it needs is unreachable), `accept` fails with the corresponding infra end-state — it **MUST NOT** default to accepting or rejecting the fragment.

**Errors:** `ingesting` — `malformed_fragment` (the fragment failed normalization or required-metadata validation), `unsupported_source` (no normalizer is registered for the fragment's source type), `model_failure` (the model call behind the relevance filter failed after the gateway's retries — the fragment is neither persisted nor rejected; retry).

**Access:** `ingest` ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

**Idempotency:** idempotent on `fragment.message_id` (key-identified), **for accepted fragments only** — a reused id of an accepted fragment is a replay: a full no-op returning the recorded `AcceptResult` (the originally minted `ingest_id` and `fragment_id`; no new event). A rejection persists nothing and leaves no replay record: re-submitting a previously rejected `message_id` is a **fresh evaluation** — the relevance filter runs again and **MAY** accept what it previously rejected (the filter is an implementation choice and its behavior may change between evaluations); each rejecting evaluation mints its own `ingest_id` and emits its own `FragmentRejected` event. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** concurrent-safe; per-fragment writes are serialized.

---

### `get`

```typescript
function get(fragment_id: FragmentId): Promise<NormalizedFragment | null>
```

Fetch a fragment with its full metadata. Returns `null` if not present — absence is `null`, never an error, matching [`Atoms.get_by_id`](./atoms.md#get_by_id). Because rejected fragments are never persisted, only ids minted on the accept path resolve; a `FragmentId` taken from an `AcceptResult` or `FragmentAccepted` event resolves until the fragment is deleted by deployment retention policy (see Notes for implementers).

**Errors:** `querying` — universal end-states only (absence is `null`, not an error).

**Access:** `query`.

---

### `list`

```typescript
function list(filter?: FragmentFilter, page?: PageRequest): Promise<Page<NormalizedFragmentMetadata>>

interface FragmentFilter {
  source_uri_prefix?:  string
  sender?:             string
  content_kind?:       ContentKind
  since?:              string   // RFC 3339
  until?:              string
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

Iterate fragment metadata, paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads) — `since` / `until` bound the timeframe, the cursor enumerates within it (timestamps are not unique and cannot resume an enumeration). Payloads are not returned by `list`; fetch them with `get`.

**Errors:** `querying` — `invalid_query` (malformed filter, or invalid paging cursor).

**Access:** `query`.

---

### `changes_since`

Shared signature from [`00-shared-types.md`](./00-shared-types.md). Event variants:

```typescript
type IngestionEvent =
  | { kind: "FragmentAccepted"; payload: { fragment_id: FragmentId; metadata: NormalizedFragmentMetadata } }
  | { kind: "FragmentRejected"; payload: { message_id: string; reason: string } }
```

`FragmentRejected` is an audit record: it references no persisted fragment (a rejected fragment has no `FragmentId`) and applies as a no-op to materialized state. Because a rejection leaves no replay record, the same `message_id` **MAY** appear in multiple `FragmentRejected` events — one per rejecting evaluation — and a later `FragmentAccepted` for that `message_id` is legal (the filter re-evaluated and accepted).

Rarely consumed in steady state; useful for telemetry and re-runs.

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible: new `ContentKind` values, new event kinds.
- Breaking: removing fields, changing idempotency semantics.
- `NormalizedFragment.extras` is free-form (`Record<string, unknown>`) by design; its contents are outside the versioned surface — keys appearing or disappearing there is not a contract change.

## Notes for implementers

- Source adapters and the relevance filter are implementation choices and are not part of this contract. The contract is "given a `NormalizedFragment`, decide accept/reject and persist accepted ones."
- Structured inputs — including OKF bundles ([Appendix B — OKF Profile](../spec/appendix-b-okf-profile.md)) — are handled by the same open, LLM-driven normalizer as any other source: they become `NormalizedFragment`s and flow through `accept` like everything else. There is **no inbound profile registry** — profiles govern only the [Projection facet's output](./projection.md#output-profile), never ingestion's input.
- The relevance filter may call out to LLM Gateway with `aala.relevance_filter`; this is observable through LLM Gateway's `usage()` API but not through Ingestion's surface.
- Tree assignment for extracted atoms happens downstream in Atoms (via extractor hints or default tree). Ingestion is tree-agnostic.
- Fragment retention and deletion are deployment policy, out of scope for this contract: `aala-v0.1` defines no purge or redaction surface. A deployment that deletes fragments out of band leaves atom `Source.uri` provenance links pointing at fragments `get` resolves to `null` — that is the defined absence signal, not corruption.
