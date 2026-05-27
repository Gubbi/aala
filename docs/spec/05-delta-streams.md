# 05 — Delta Streams

Every stateful container MUST expose an ordered change stream via `changes_since(ref)`. This chapter defines the binding semantics.

## Universal signature

```typescript
function changes_since(ref: Ref | null, limit?: number): Promise<ChangesPage<TVariant>>

interface ChangesPage<TVariant> {
  events:     ChangeEvent<TVariant>[]
  next_ref?:  Ref       // absent when the caller has caught up
}

interface ChangeEvent<TVariant> {
  ref:         Ref      // monotonic checkpoint after applying this event
  kind:        string   // event kind, e.g., "Added", "StatusChanged"
  payload:     TVariant
  emitted_at:  string   // ISO 8601
  source_ref?: Ref      // origin checkpoint when the event was caused by an upstream delta
}
```

`ref = null` MUST return events from the beginning of the current snapshot's stream. The `limit` parameter is a soft hint; implementations MAY return fewer events.

## Ordering

Events within a single container's stream MUST be **totally ordered**. For any two events `e1` and `e2` in the stream, exactly one of `e1.ref < e2.ref` or `e1.ref > e2.ref` MUST hold (refs are comparable monotonic identifiers).

Total ordering means: a consumer reading the stream sequentially MUST observe events in the same order any other consumer reading the same stream observes them.

Events across containers are NOT totally ordered with each other. A consumer that subscribes to multiple containers MUST track an independent ref per container.

## Replay equivalence

For any two refs `X` and `Y` in the same container's stream with `X < Y`:

> `snapshot(X) + apply(deltas(X → Y)) ≡ snapshot(Y)`

In other words: applying the deltas between `X` and `Y` to the state at `X` MUST produce the state at `Y`. This invariant is what permits consumers to maintain derived state by consuming the stream incrementally rather than by full rebuilds.

A consumer that loses its checkpoint MAY rebuild from scratch by:

1. Reading the snapshot's full state (via the relevant container's read APIs).
2. Recording the latest ref at the time of the read.
3. Subscribing to `changes_since(ref)` from that point forward.

## Paging

A `ChangesPage` returned with `next_ref` present indicates more events are available. The caller MUST pass `next_ref` as the next call's `ref` to continue.

A `ChangesPage` returned without `next_ref` indicates the caller has caught up to the current head. Subsequent calls with the same `ref` MAY return empty pages until new events are emitted.

Implementations MAY enforce a maximum page size; the cap is not part of this contract. A consumer MUST keep paging until `next_ref` is absent if it wants to reach the head.

## Event semantics

Each container's interface contract enumerates its event variants. The full list of event kinds and their payload shapes is normative per container; see:

- [Ingestion](../interfaces/ingestion.md#changes_since)
- [Atoms](../interfaces/atoms.md#changes_since)
- [Projection](../interfaces/projection.md#changes_since)
- [Hierarchical Navigation](../interfaces/hierarchical-nav.md#changes_since)
- [Blast Radius](../interfaces/blast-radius.md#changes_since)
- [Synthesis](../interfaces/synthesis.md#changes_since) (largely empty for stateless impls)
- [Quality](../interfaces/quality.md#changes_since)
- [Orchestration](../interfaces/orchestration.md#changes_since)

Implementations MAY emit additional event kinds (for impl-specific features) provided that the standard kinds are present and unchanged.

## Cross-snapshot behavior

A consumer's `ref` is meaningful within a single snapshot. Calling `changes_since(ref)` against a ref that belongs to a different snapshot — or to a discarded snapshot — MUST result in either:

- A clean `Conflict` error indicating the snapshot has changed, OR
- A re-baseline page that delivers the full reachable state of the current snapshot followed by `next_ref` to continue from the current head.

The choice is implementation-specific; both are conformant.

When `Orchestration.switch_to(new_snapshot)` runs, consumers MUST be able to detect the switch and re-baseline. Implementations MAY provide a `snapshot_id` field on `ChangeEvent` so consumers can detect transitions, OR rely on the consumer subscribing to `Orchestration.changes_since` for `SnapshotSwitched` events.

## Producer obligations

A container MUST emit a delta-stream event for every observable state change in its container. "Observable state change" means a change that another container or external caller could detect by re-reading the same data through the container's read API.

Internal optimizations (e.g., rebalancing a cache, compacting an index) MAY proceed without emitting events, provided the externally-observable state is unchanged.

## Consumer obligations

A consumer subscribing to a container's stream:

- MUST track its checkpoint `ref` durably enough to survive its own restarts (or be prepared to replay from scratch).
- MUST be tolerant of unknown event kinds (forward compatibility with patch-compatible additions per [11 — Versioning](./11-versioning.md)). Unknown kinds SHOULD be ignored.
- MUST apply events in order. Skipping or reordering events violates the consumer's derived-state invariants.

## Idempotent consumption

Some consumers MAY observe a delta event more than once (e.g., across a restart). Consumers MUST be prepared to handle repeated events. The recommended pattern: each event's `ref` is a monotonic checkpoint; a consumer that has applied a ref `R` MUST treat any subsequent event with `ref ≤ R` as a no-op.

Producer implementations SHOULD NOT emit duplicate events with the same `ref` value; but consumers MUST be prepared for it regardless (e.g., across consumer-side retries).
