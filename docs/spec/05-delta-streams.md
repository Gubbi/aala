# 05 — Delta Streams

Every stateful container MUST expose an ordered Delta Stream via `changes_since(ref)` and its cold-start companion `as_stream()`. This chapter defines the binding semantics.

## Universal surface

Both stream methods — `changes_since(ref, limit?)` for incremental deltas and `as_stream(ref?, limit?)` for cold-start bootstrap — are declared once, together with the `ChangeEvent` / `ChangesPage` envelope, in [`interfaces/00-shared-types.md` § Delta-Stream events](../interfaces/00-shared-types.md#delta-stream-events); every stateful container implements them. The shared declaration owns the shapes and signatures ([01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership)); this chapter owns their semantics.

Two envelope fields answer different questions and are easily conflated. `source_ref` is **positional and stream-pair-local**: the exact upstream checkpoint that triggered this event, so a consumer can align two adjacent streams and resume coherently across one producer→consumer hop. `correlation_id` is **causal-origin and global**: a stable id for the operation that ultimately caused the event, identical across every container in the cascade.

The `ref` **argument** is the caller's cursor — the checkpoint it last consumed; the call returns the events that occurred *after* it. This argument is distinct from each `ChangeEvent.ref` (the per-event checkpoint, which is always present): a caller pages by taking the page's `next_ref` (or the last event's `ref`) and passing it back as the `ref` argument on the next call, until `next_ref` is absent (caught up).

`ref = null` means the caller has no cursor yet and MUST return events from the **beginning of the current working snapshot's own Delta Stream** — i.e. the actual events from the operations that occurred in *this* working snapshot (the changes layered on top of its parent). The stream contains only real operation events; it is the post-fork delta, **not** a reconstruction of the inherited parent state. A consumer that needs to build the full current state from nothing uses [`as_stream()`](#full-state-stream) instead. The `limit` argument is a soft hint on page size; implementations MAY return fewer events.

## Full-state stream

A consumer with **no materialized state** for this snapshot's lineage (a brand-new or state-wiped consumer — a fresh external mirror, an index being built for the first time) bootstraps the entire current state through `as_stream()`.

`as_stream` emits the **bootstrap sequence**: an ordered event sequence that, applied to empty, reconstructs the container's full current-snapshot state. The sequence is a synthetic prefix followed by a real suffix:

- **Synthetic prefix** — the parent canonical's materialization, replayed as synthetic creation events. Equivalence is structural (see [Replay equivalence](#replay-equivalence)), so one synthetic event per currently-reachable record is sufficient — for the Atoms container, one per asserted atom, live and tombstone alike; derived atoms are never replayed ([Derived atoms in the stream](#derived-atoms-in-the-stream)) — and intermediate historical churn is not replayed. The prefix is drawn from immutable state: a canonical snapshot never changes.
- **Real suffix** — the events of this working snapshot's own Delta Stream, in stream order, carrying their real refs.

The synthetic prefix's internal order is producer-chosen and MUST be stable for the snapshot's lifetime; together with the append-only suffix, this makes every bootstrap cursor exactly resumable — a resumed bootstrap delivers every remaining event exactly once.

**Paging and resumption.** A bootstrap pages exactly like `changes_since`: the first call passes no `ref` (start from empty); each returned page's `next_ref` is a **bootstrap cursor** that the consumer passes back to `as_stream` to fetch the next page; a page without `next_ref` completes the bootstrap. The producer MUST accept any ref it delivered through `as_stream` as a bootstrap cursor and resume the sequence immediately after the position it names, across any time gap within the snapshot's lifetime ([Stream retention](#stream-retention)). Bootstrap cursors are snapshot-scoped like every ref ([Cross-snapshot behavior](#cross-snapshot-behavior)).

**Continuing incrementally.** After the bootstrap completes, the consumer's cursor for `changes_since` is the `ref` of the **last event the bootstrap delivered**; a bootstrap that delivered no events (empty container state) continues with `changes_since(null)`. To make this seam well-defined, synthetic refs are valid `changes_since` cursors: presented to `changes_since`, a synthetic ref positions the consumer **before the first event of the snapshot's own Delta Stream** — every synthetic event reconstructs inherited state, which precedes every post-fork delta. (A real ref delivered in the suffix is an ordinary stream checkpoint and needs no special rule.)

**Concurrent writes during a bootstrap.** A bootstrap spanning multiple pages never yields a torn state: the synthetic prefix is drawn from the immutable parent canonical, and the real suffix is append-only — operations that run while the consumer is paging append events that later pages deliver in stream order. The bootstrap completes when the consumer catches up with the head as of its final page.

**Completion is the consistency point.** State assembled from a bootstrap is consistent only once the bootstrap completes (a page without `next_ref`). A consumer MUST NOT treat a partially-consumed bootstrap as usable state, and MUST NOT abandon a bootstrap for `changes_since` mid-prefix — a mid-prefix synthetic ref skips the remainder of the inherited state.

Choosing between the two:

- **`changes_since(ref | null)`** — when you already hold materialized state for this snapshot's lineage and want only its deltas. `null` starts at the beginning of this working snapshot's own operations. This is the normal path for a derived container, which inherits its parent's materialized state on fork (see [04 — Snapshots](./04-snapshots.md)) and only needs the post-fork deltas.
- **`as_stream()`** — when you hold nothing and need the full current state rebuilt. Cold start only.

A caller that merely wants to *inspect* current state reads it directly (the read APIs); neither stream method is for interactive reads.

## Ordering

Events within a single container's stream MUST be **totally ordered**: the stream is a single sequence, each event occupies exactly one position in it, and two distinct events MUST NOT carry the same `ref`. Each event's `ref` names the checkpoint reached after applying it.

The order is held by the **producer**; refs are opaque ([01 — Conventions § Terminology](./01-conventions.md#terminology); format in [shared-types § Identifiers](../interfaces/00-shared-types.md#identifiers)). A `Ref` value carries no client-computable order: consumers MUST NOT compare two refs to determine stream position — no lexicographic, numeric, or any other value-derived comparison. The only client-side operation on refs is **equality**. Positioning is server-side: a consumer places itself in the stream exclusively by passing a previously delivered ref back to `changes_since` (or `as_stream`), which returns events strictly after the named checkpoint.

Total ordering means: a consumer reading the stream sequentially MUST observe events in the same order any other consumer reading the same stream observes them.

Events across containers are NOT totally ordered with each other. A consumer that subscribes to multiple containers MUST track an independent ref per container.

## Causal ordering within a stream

Within a single container's stream, events MUST respect causality:

- `Added(A)` MUST precede any event referencing `A` (e.g., `StatusChanged(B, hanging, cause=A)`).
- `Updated(A)` MUST follow `Added(A)`; successive `Updated(A)` events appear in revision order, each `before` equal to the prior `after`.
- Cascade events emitted as consequences of one source transition MUST appear in the deterministic per-channel order defined in [07 — Atom Lifecycle](./07-atom-lifecycle.md); each cascaded `StatusChanged` MUST follow the event recording the transition that caused it (the transition of its `cause_atom_id`).
- `DerivedInvalidated` events MUST appear after the source-atom transition event that triggered them; derived-atom events caused by a content revision's entailment recompute MUST follow the `Updated` event that caused them.
- A `DerivedAdded` event MUST follow the `Added` events for every atom in the derived atom's `derivation.from`. (Derived-atom events are advisory and apply as no-ops; [Derived atoms in the stream](#derived-atoms-in-the-stream).)

Sole exception to the first rule: a `Rejected` event whose payload carries `candidate` records the Conflict pipeline's rejection of a **never-committed** candidate ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)). Its `atom_id` has no prior `Added`; the event is an audit record and MUST be applied as a no-op to materialized state.

A consumer who applies events in stream order MUST never observe a state with non-existent dependencies.

**Content revisions in the stream.** An `Updated` event records an in-place revision of one existing atom ([03 — Data Model § Mutability and revision](./03-data-model.md#mutability-and-revision)): the same `id`, with full `before` / `after` payloads. A consumer maintaining materialized state applies it as a whole-atom replace. `Updated` never implies a status change — status transitions are recorded by their own events — and a revision triggers no cascade, so an `Updated` event is never followed by cascade consequences of its own ([07 § Cascade rules](./07-atom-lifecycle.md#cascade-rules)).

## Correlation across streams

Every event carries a `correlation_id` identifying the operation that caused it. It is an `ingest:`-, `request:`-, or `system:`-prefixed identifier; its value is opaque beyond the prefix and exists to support **grouping and debugging** — reconstructing everything one ingest, API request, or internal job set in motion across containers.

- **Stamping.** The initiating context sets it: `Ingestion.accept(fragment)` stamps the `ingest:<opaque>` id; direct API-initiated writes stamp the `request:<opaque>` id assigned by [Orchestration](../interfaces/orchestration.md); a container that starts an internal operation (periodic rebuild, integrity sweep, maintenance pass) stamps its own `system:<opaque>` id.
- **Propagation invariant.** When a container processes an upstream event or input carrying a `correlation_id`, it MUST copy that same `correlation_id` onto every event it emits in response. The id therefore reaches every container in a causal cascade unchanged — e.g. one ingest's `FragmentAccepted`, the Atoms `Added` / cascade events it drives, and the Atoms projection-facet `DocChanged` events all share it.
- **Contiguous within a snapshot.** Within a single snapshot's stream, operations are serialized at operation grain (see [04 — Snapshots](./04-snapshots.md) § Operation isolation), so all events sharing a `correlation_id` form a **contiguous, causally-ordered run** — two operations' events never interleave within that snapshot. This is what keeps each ingest's changes a clean, reviewable run even when several small ingests share one uncommitted snapshot. Concurrency across operations comes from concurrent sessions on separate snapshots, not interleaving.
- **Correlation-aligned batching (fan-in).** A derived consumer (e.g. the projection facet) MAY coalesce multiple upstream events into one emitted event **only when they share a `correlation_id`**; a batch MUST NOT cross `correlation_id` boundaries. Since upstream events are already contiguous per operation within a snapshot, this aligns with the structure the upstream already produces — the consumer batches one operation's run at a time. The only cost is forgoing *cross-operation* render coalescing (a document touched by two operations re-renders once per operation; deterministic rendering + the render cache keep this cheap). Consequently **every event carries exactly one `correlation_id`** — a derived event inherits the single id of the operation it reflects, and internally-initiated work (periodic rebuilds, integrity sweeps, maintenance) is itself an operation with its own `system:<opaque>` id. There is no uncorrelated event.
- **A join key across snapshots and containers.** Across different snapshot streams and across containers (refs are per-stream, not co-ordered), `correlation_id` is the filter that stitches one operation's fan-out back together — query each container's stream filtered by it and union.
- **Not an ordering or replay input.** `correlation_id` is metadata for correlation only; it MUST NOT affect ordering, replay equivalence, or any logical consequence.

**Reconstructing an operation trace.** Because every operation-caused event carries exactly one `correlation_id` — contiguous within a snapshot, recoverable by filter across snapshots and containers — the full set of effects of one ingest or request is reconstructed by filtering each container's stream by that `correlation_id` and unioning the results. No central trace service or multi-valued event metadata is required; `source_ref` supplies the finer causal edge (the upstream checkpoint a derived event consumed through) within that grouping.

## Replay equivalence

For any two checkpoints `X` and `Y` in the same container's stream, with `X` preceding `Y` in stream order:

> `snapshot(X) + apply(deltas(X → Y)) ≡ snapshot(Y)`

In other words: applying the deltas between `X` and `Y` to the state at `X` MUST produce the state at `Y`. This invariant is what permits consumers to maintain materialized state by consuming the stream incrementally rather than by full rebuilds.

Equivalence is structural and covers **asserted state**: the consumer's record set — for the Atoms container, the asserted atoms, their statuses, and RelationAtoms — and the `ChangeEvent`s themselves MUST match. Derived atoms sit outside the invariant: they are a deterministic function of asserted state ([04 — Snapshots § Derived atoms within a snapshot](./04-snapshots.md#derived-atoms-within-a-snapshot)), and a consumer that needs them recomputes them from its replayed asserted state ([Derived atoms in the stream](#derived-atoms-in-the-stream)). Internal indices and caches likewise need not match (they are recomputable from the same axioms).

A consumer that has lost all its state and must rebuild from scratch calls `as_stream()` (see [Full-state stream](#full-state-stream)): a single ordered stream from empty to a consistent head; completing the bootstrap yields exactly the cursor to continue from with `changes_since` (the final event's ref). This bootstraps through the same delta channel — cold-start and steady-state share one event-handling path, with no separate bulk-export endpoint and no read-then-record-ref race.

A consumer that still holds materialized state for this snapshot's parent does **not** rebuild — it applies only this snapshot's deltas (`changes_since(null)`) on top of its inherited state.

## Paging

A `ChangesPage` returned with `next_ref` present indicates more events are available. The caller MUST pass `next_ref` as the next call's `ref` to continue — back to the method that returned it: a bootstrap pages through `as_stream`, an incremental read through `changes_since`.

A `ChangesPage` returned without `next_ref` indicates the caller has caught up to the current head (for `as_stream`: the bootstrap is complete). Subsequent calls with the same `ref` MAY return empty pages until new events are emitted.

Implementations MAY enforce a maximum page size; the cap is not part of this contract. A consumer MUST keep paging until `next_ref` is absent if it wants to reach the head.

## Stream retention

A snapshot's Delta Stream MUST be retained un-truncated for the snapshot's lifetime: every ref the snapshot's stream methods have delivered — real and synthetic alike — remains a resumable cursor until the snapshot is discarded ([Appendix A § Identifier minting](./appendix-a-registries.md#identifier-minting)). Implementations MUST NOT compact or truncate a stream within a snapshot's lifetime; the only invalid-ref condition is `stale_ref` ([Cross-snapshot behavior](#cross-snapshot-behavior)). Bounding stream growth is a snapshot-lifecycle concern — a deployment discards snapshots it no longer needs — never a stream-truncation concern.

## Event semantics

Each container's interface contract enumerates its event variants. The full list of event kinds and their payload shapes is normative per container; see:

- [Ingestion](../interfaces/ingestion.md#changes_since)
- [Atoms](../interfaces/atoms.md#changes_since)
- [Projection](../interfaces/projection.md#changes_since)
- [Hierarchical Navigation](../interfaces/hierarchical-nav.md#changes_since)
- [Blast Radius](../interfaces/blast-radius.md#changes_since)
- [Quality](../interfaces/quality.md#changes_since)
- [Orchestration](../interfaces/orchestration.md#changes_since)

Implementations MAY emit additional event kinds (for impl-specific features) provided that the standard kinds are present and unchanged.

## Cross-snapshot behavior

A consumer's `ref` is meaningful only within a single snapshot. Calling `changes_since(ref)` or `as_stream(ref)` with a ref that does not name a checkpoint the calling session's bound snapshot delivered — a ref from another snapshot, from a discarded snapshot, or one the producer does not recognize — MUST fail with a `querying` / `stale_ref` Activity Error (classification per the registry in [10 — Error Model](./10-error-model.md#activity-end-states)) indicating the snapshot has changed. Implementations MUST NOT silently re-baseline the caller onto the bound snapshot.

On a `stale_ref` error, the consumer re-baselines **explicitly**, choosing by what it still holds:

- If it has materialized state for the bound snapshot's parent (compare `SnapshotRef.parent` against the snapshot it last materialized), it applies the bound snapshot's own deltas on top — `changes_since(null)`.
- Otherwise it cold-starts via `as_stream()`, completing the bootstrap and continuing incrementally from its final event ([Full-state stream](#full-state-stream)).

Rebinding is an act of the session itself (`Orchestration.switch_snapshot`; [04 — Snapshots § Session binding](./04-snapshots.md#session-binding)), so the caller that rebinds knows synchronously, and no event records it. The `stale_ref` error is the contract backstop: consumer logic still holding a cursor from the previous binding discovers the rebind on its next `changes_since(ref)` call and re-baselines by one of the two paths above.

## Producer obligations

A container MUST emit a Delta-Stream event for every observable state change in its container, with one carve-out for derived atoms (next section). "Observable state change" means a change that another container or external caller could detect by re-reading the same data through the container's read API.

Internal optimizations (e.g., rebalancing a cache, compacting an index) MAY proceed without emitting events, provided the externally-observable state is unchanged.

## Derived atoms in the stream

The stream's replayable contract covers asserted state ([Replay equivalence](#replay-equivalence)). Derived atoms ([03 — Data Model § Entailment](./03-data-model.md#entailment)) are a deterministic function of that state, and which of them are materialized at any moment is a per-snapshot caching choice ([04 — Snapshots § Derived atoms within a snapshot](./04-snapshots.md#derived-atoms-within-a-snapshot)) — so the stream does not carry derived state:

- **Consumers recompute.** A consumer that needs derived state runs the entailment rules over its replayed asserted state. Everything those rules read — predicate schemas, chain declarations, the asserted edges — arrives through the asserted events.
- **Derived-atom events are advisory.** `DerivedAdded` and `DerivedInvalidated` ([`interfaces/atoms.md` § `changes_since`](../interfaces/atoms.md#changes_since)) report the producer's materialization activity: `DerivedAdded` records that a derived atom was materialized into snapshot state; `DerivedInvalidated` records a channel-3 invalidation of a materialized derived atom ([07 — Atom Lifecycle § Channel 3](./07-atom-lifecycle.md#channel-3-derivation-invalidation)). Both apply as **no-ops** to replayed asserted state. Consumers MAY use them as recompute hints; they MUST NOT depend on them for correctness — a lazily caching producer materializes nothing and emits neither.
- **Emission follows materialization, and is coalescible.** A producer SHOULD emit `DerivedAdded` when a derived atom is materialized and `DerivedInvalidated` when a materialized derived atom is invalidated; it MAY coalesce or omit high-volume derived-atom events (e.g. a closure recomputation touching many derived edges), provided the asserted events are complete. Where nothing is materialized, the logical obligations of channel 3 are met by recomputation at read time and no event fires.
- **Derived atoms never ride state-carrying kinds.** When a producer records derived-atom activity it MUST use the derived-atom event kinds: never `Added` for a derived atom's materialization, never `StatusChanged` for its invalidation ([07 — Atom Lifecycle § Cascade provenance emissions](./07-atom-lifecycle.md#cascade-provenance-emissions)). This keeps the asserted replay clean regardless of caching strategy.

Lazy and eager implementations therefore lawfully emit different derived-atom events over the same asserted history — which is exactly why these events sit outside replay equivalence.

## Consumer obligations

A consumer subscribing to a container's stream:

- MUST track its checkpoint `ref` durably enough to survive its own restarts (or be prepared to replay from scratch).
- MUST be tolerant of unknown event kinds (forward compatibility with backward-compatible additions per [11 — Versioning](./11-versioning.md)). Unknown kinds SHOULD be ignored.
- MUST apply events in order. Skipping or reordering events violates the consumer's derived-state invariants.

## Idempotent consumption

Redelivery happens: a consumer that applies events and restarts before durably advancing its cursor re-issues `changes_since` from its older durable cursor and receives the already-applied suffix again; a retried page delivers the same events twice. Consumers MUST tolerate redelivery. Two conformant patterns:

- **Atomic checkpointing** — persist the cursor in the same transaction as the state changes it covers; a restart then resumes exactly where application stopped, and redelivery never overlaps applied state.
- **Idempotent application** — event payloads carry whole values (the full atom in `Added` and `Updated.after`, absolute statuses in `StatusChanged`), so re-applying an event to state that already reflects it converges to the same state.

Redelivery detection needs no ref arithmetic: refs are opaque and order-incomparable ([Ordering](#ordering)). A consumer MAY skip an event whose `ref` exactly equals one it has already applied — it needs to remember only the refs seen since its last durable checkpoint, because the producer returns only events after the cursor it is given.

Producers MUST NOT emit two **distinct** events sharing a `ref` ([Ordering](#ordering)). Delivery of the **same** event more than once — producer-side retries included — is permitted; consumers handle it by the rules above.
