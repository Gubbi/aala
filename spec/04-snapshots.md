# 04 — Snapshots

A **snapshot** is the unit of coherence across containers. This chapter defines its lifecycle and the invariants every conformant implementation must honor.

## Definition

A snapshot is an addressable, coherent version of all snapshot-bound state — atoms, projections, blast reports, navigation trees, derived-atom caches, and any other state a container persists. A snapshot is identified by an opaque `SnapshotId`.

The realization is implementation-specific: a snapshot MAY be a git ref + working tree, a row in a versioning table, a content-addressed manifest, an immutable in-memory structure, or anything else with the right semantics. The contract is the behavior described in this chapter, not the realization.

## States

Every snapshot is in exactly one of three states:

| State | Meaning |
|---|---|
| `working` | Mutable. Writes execute against it through sessions bound to it ([Session binding](#session-binding)). |
| `canonical` | Immutable. Represents committed state. Reads against canonical snapshots **MUST** be stable and reproducible. The deployment's single **canonical pointer** designates which canonical snapshot is *the* canonical ([Canonicalization](#canonicalization)). |
| `discarded` | Working snapshots not promoted to canonical. Eligible for garbage collection at the implementation's discretion. |

## Transitions

Allowed transitions:

- (none) → `working` — via `Orchestration.fork()`.
- `working` → `canonical` — via `Orchestration.publish_as_canonical(id)`, or the externally-managed equivalent ([Canonicalization](#canonicalization)).
- `working` → `discarded` — via `Orchestration.discard(id)`.
- `canonical` → (none) — canonical snapshots MUST NOT be discarded by the implementation. They MAY be archived per implementation policy, but their identity and reachability MUST be preserved for as long as referenced — designated by the canonical pointer, or reachable as an ancestor of any non-discarded snapshot.
- `discarded` → (none) — implementations MAY garbage-collect discarded snapshots.

`canonical → working` is **NOT** a valid transition. To work from a canonical snapshot, a caller forks a new working snapshot from it.

## Session binding

Every call to a capability container executes within a **session** — the perimeter context a deployment establishes for a caller. How a session is established and conveyed (an HTTP token, an MCP connection, a CLI working directory) is wire-profile- and deployment-specific; the binding contract below is normative regardless of mechanism. Establishment is also where a deployment attaches caller identity and resolves it to the session's **access grants** — the grant model is bound in [02 — Conformance § Access control](./02-conformance.md#access-control); identity itself stays outside the contract. Establishment is likewise the multi-tenancy seam: an operator running one logical deployment per tenant ([00 — Introduction § What's out of scope](./00-introduction.md#whats-out-of-scope)) routes each caller to its deployment here, and — like identity — that routing stays outside the contract.

- A session binds **exactly one snapshot at a time** — its **bound snapshot**. Throughout the spec and interfaces, "the current snapshot" means the calling session's bound snapshot. `Orchestration.current_snapshot()` returns it.
- The binding is the **sole snapshot selector**: every read and write a session issues executes against its bound snapshot. No interface method takes a per-call snapshot parameter to address a different snapshot ([Cross-snapshot reads](#cross-snapshot-reads)).
- `Orchestration.switch_snapshot(snapshot_id)` atomically rebinds **the calling session only**; other sessions' bindings are unaffected.
- The initial binding is determined at session establishment. Absent an explicit choice by the establishment mechanism, the session binds to the snapshot the canonical pointer designates ([Canonicalization](#canonicalization)).
- A session **MAY** be bound to a `working` or a `canonical` snapshot. Binding to a canonical snapshot is the mechanism for stable historical reads. Write entry points require a `working` binding: a write issued in a session bound to a `canonical` snapshot **MUST** fail with `snapshotting` / `canonical_protected` ([10 — Error Model § Session-binding failures](./10-error-model.md#activity-end-states)).
- A session **MUST NOT** be bound to a `discarded` snapshot: `switch_snapshot` naming one fails with `snapshotting` / `discarded`, and when a session's bound snapshot is discarded out from under it, every subsequent call in that session **MUST** fail with `snapshotting` / `discarded` until the session rebinds.

A call executes wholly against the snapshot its session was bound to when the call was admitted; `switch_snapshot` takes effect for calls admitted after it returns. Partial execution of one call across two snapshots is **NOT** permitted.

Concurrent work is concurrent sessions: callers operate on different snapshots by holding different sessions, each bound to its own snapshot — never by per-call snapshot selection ([Operation isolation and concurrency](#operation-isolation-and-concurrency)).

Writes land in the bound working snapshot **immediately**: each committed operation (ingest, proposal, revision, status change) appends to that snapshot's Delta Stream and becomes part of its reachable state at once, so `changes_since` against that snapshot reflects it right away. This is distinct from **publishing**. An ingest is "committed to the working snapshot" — queryable now via reads and `changes_since` — well before it is "published to canonical" via `publish_as_canonical` (the separate, atomic promotion to immutable history). There is no separate staging area: the working snapshot is where in-progress work lives, and canonical receives it only at publish time. (Writing to canonical is impossible — to edit from a canonical baseline, fork a working snapshot first.)

## Coherence invariant

Cross-container reads at a single snapshot identifier MUST be coherent. Specifically:

> For any two reads `r1` and `r2` issued against the same `SnapshotId`, the state observed by `r1` and `r2` MUST be consistent with a single point in the snapshot's history. No interleaving with concurrent writes against that snapshot MAY produce a state that is not reachable by sequencing the writes.

Implementations achieve this via any of:

- Single-writer-per-snapshot with snapshot-isolated reads.
- Multi-version concurrency control (MVCC) with a stable read transaction.
- Append-only logging with derived projections built deterministically from the log.

The contract is the property, not the technique.

## Operation isolation and concurrency

The unit of serialization against a snapshot is the **operation** — a single `ingest` / `propose` / `update` / `update_status`, including the conflict classification, cascade to fixpoint, and entailment recompute it triggers. (`BlastRadius.analyze` is read-only and runs outside the write operation; a blast-driven resolution reaches the snapshot as an ordinary `update_status` operation.)

- **Per-operation serialization.** Within a snapshot, operations are serialized: one operation's effects commit as a unit before the next begins; concurrent operations against the same snapshot are queued or rejected with a `contended` Activity Error (`infra`/`retryable`), never interleaved. Consequently each operation's events form a contiguous, causally-ordered run in that snapshot's [Delta Stream](./05-delta-streams.md) — so several small ingests sharing an uncommitted snapshot (e.g. near-real-time message-by-message capture) each remain a clean, reviewable run rather than being mixed together.
- **Concurrency comes from sessions on separate snapshots, not intra-snapshot interleaving.** A large or batch document import SHOULD run in its own working snapshot — fork, bind a session to it, build the import incrementally, review the coherent diff, publish atomically — so its processing neither contends with nor interleaves other work. Small incremental ingests MAY share a working snapshot — including from multiple sessions bound to the same snapshot — serialized per operation.
- **Atomic visibility of a whole import is the publish boundary**, not stream ordering: a working snapshot is promoted to canonical atomically (`publish_as_canonical`), so canonical readers see an import all-at-once or not at all.

These guarantees are backend-independent. A "snapshot" may be a git branch, an MVCC version in a relational/document store, a versioned namespace in a shared store, or a content-addressed manifest; "publish" is an atomic version promotion; per-operation serialization is the backend's serializable-transaction or lock scope. The contract is the property, not the technique.

## Derived atoms within a snapshot

A snapshot contains both asserted atoms (those with `derivation` absent) and derived atoms (those with `derivation` populated). For coherence:

- Derived atoms MUST be consistent with their source atoms within the same snapshot. If a derivation source is in the snapshot, its derivatives are present (eager / hybrid caching) or computable (lazy caching).
- Two snapshots MAY differ in which derived atoms are materialized — caching strategy is per-snapshot. The set of derived atoms reachable through computation MUST be identical given the same asserted set.

## Lineage

A snapshot MAY have a parent (the snapshot it was forked from). Lineage MUST be preserved for as long as either the snapshot or any of its descendants exist.

`SnapshotRef.parent` returns the parent's `SnapshotId` (or absent for root snapshots).

A fork **inherits the parent's entire snapshot-bound state** across every container — atoms, projections, navigation trees, blast reports, derived-atom caches — copy-on-write. The new working snapshot's own Delta Stream begins empty and accumulates only the operations performed in it; `changes_since(null)` on it returns exactly those post-fork deltas (the parent state is the implicit base). A derived container therefore does not rebuild on fork: it carries its parent-snapshot materialization forward and updates it incrementally as the working snapshot's atoms change (e.g. the projection facet re-renders only affected sections). Full reconstruction from empty — for a consumer holding no inherited state — is the explicit `as_stream()` path (see [05 — Delta Streams](./05-delta-streams.md) § Full-state stream), which replays the parent canonical's materialization followed by this snapshot's deltas.

`Orchestration.list_snapshots({ parent_of: id })` MUST return all snapshots forked from `id`.

## Canonicalization

### The canonical pointer

A deployment has **exactly one canonical pointer** — the designation of its single current canonical snapshot. `canonical` as a snapshot *state* marks immutable published history, and any number of snapshots accumulate in it over time; the pointer designates which of them is *the* canonical: the default fork source, the default initial session binding, and the baseline the next publish must descend from. When no snapshot has been published yet, the pointer is unset and the first publish establishes it.

### Publish is a compare-and-swap

`Orchestration.publish_as_canonical(id)` moves the canonical pointer by an **atomic compare-and-swap**. The publish **MUST** succeed iff it is a **fast-forward**: the snapshot the pointer currently designates is an ancestor of the published snapshot (reachable from it through the `parent` chain), or the pointer is unset. On success — atomically — the published snapshot's state becomes `canonical`, the pointer moves to it, and a `SnapshotPublished` event records the movement. The previously canonical snapshot keeps its `canonical` state: it remains immutable history; only the pointer moves off it.

If the canonical pointer has moved since the published snapshot's lineage left it — another session published first — the call **MUST** fail with a `snapshotting` / `non_fast_forward` Activity Error (`client` / `user_action`; registry row in [10 — Error Model](./10-error-model.md#activity-end-states)). The recovery path is **re-fork and replay**: fork a new working snapshot from the new canonical and re-apply the work — e.g. re-ingest the fragments, with idempotency on `message_id` making replay safe. **Snapshot merge is out of scope for `aala-v0.1`**: no merge operation is defined, and `non_fast_forward` recovery never silently merges.

Publish is idempotent at the pointer: a publish naming the snapshot the pointer already designates is a no-op success, returning that snapshot's `SnapshotRef` — so a caller retrying a timed-out publish cannot misread its own success as a failure.

### Mechanisms

The compare-and-swap is realized by one of two mechanisms:

1. **aala-managed:** `Orchestration.publish_as_canonical(id)` executes the compare-and-swap itself and emits the `SnapshotPublished` event.

2. **Externally-managed:** the canonical pointer is realized by an external system (e.g. a git branch head moved by the user's merge or push). Where the implementation can drive that mechanism (e.g. a fast-forward push), `publish_as_canonical` executes the compare-and-swap through it, with the same contract — including `non_fast_forward` when the external system rejects a non-fast-forward promotion. Where the implementation cannot drive it (promotion happens only by external action), `publish_as_canonical` is not wired in that deployment (universal `not_wired`); the implementation observes the external pointer movement and emits `SnapshotPublished` accordingly.

Both are conformant. The published `Orchestration.capabilities()` declaration SHOULD indicate which mechanism the implementation uses.

After publication, the snapshot's state MUST be `canonical` from all containers' perspectives, and reads of that snapshot MUST be stable.

## Switching

`Orchestration.switch_snapshot(snapshot_id)` **MUST** atomically rebind the calling session ([Session binding](#session-binding)). Because Delta-Stream refs are meaningful only within one snapshot, a `changes_since(ref)` call issued after the rebind with a ref from the previously bound snapshot **MUST** fail with a `querying` / `stale_ref` Activity Error (see [05 — Delta Streams](./05-delta-streams.md) § Cross-snapshot behavior). The consumer then re-baselines: if it still holds materialized state for the newly bound snapshot's parent it applies that snapshot's deltas (`changes_since(null)`) on top; otherwise it cold-starts via `as_stream()`. A consumer that kept a per-snapshot ref can resume without re-baselining when its session rebinds back: refs remain resumable for the snapshot's lifetime ([05 — Delta Streams § Stream retention](./05-delta-streams.md#stream-retention)).

In-flight calls at the moment of rebind complete against the binding in effect at their admission ([Session binding](#session-binding)); partial execution of one call across two snapshots is **NOT** permitted.

## Removal outcomes and snapshots

When a **previously-committed** atom transitions to a removal outcome (`superseded` / `duplicate` / `removed`):

- The transition is committed and the atom is **retained as a tombstone** with the removal-outcome state — it is not deleted directly. It still appears in reads that include removal states, carrying its full removal record (`status.reason`, `status.correlation_id`, `status.meta`).
- Physical deletion happens only via the optional **garbage-collection** step (see [07 — Atom Lifecycle](./07-atom-lifecycle.md) § Garbage collection), which runs later and emits a `Purged` event.

A **never-committed** proposal (`rejected`, or one dropped by a proposal-time `duplicate` / hard-block) is not tombstoned — it was never part of the snapshot; only its `Rejected` audit event records the outcome ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).

**Tombstones are visible from transition through commit.** Within the working snapshot where the removal transition occurred, the tombstone MUST remain present — never GC-purged — from the transition until that snapshot is published or discarded. The reviewable diff of a working snapshot therefore always shows a committed atom's removal as an explicit tombstone, never as a silent disappearance. GC eligibility begins only after publication; post-publication retention, scheduling, and compaction are implementation-specific ([07 — Atom Lifecycle § Garbage collection](./07-atom-lifecycle.md#garbage-collection)).

Crucially: the atom MUST remain reachable in the **parent** snapshot at the time of the transition. Snapshot history is the audit trail; tombstoning or GC-purging an atom in snapshot `S` MUST NOT affect its presence in any ancestor snapshot.

## Atom identity across snapshots

An atom's `AtomId` is stable across snapshots. The same logical atom MUST keep the same `id` through any `Updated` event and across any number of snapshot transitions. An atom that was in snapshot `S` and is in snapshot `S+1` MUST have the same `id` in both.

A content revision ([03 — Data Model § Mutability and revision](./03-data-model.md#mutability-and-revision)) is recorded as an `Updated` event against the same `id`; the working snapshot's diff against its parent therefore shows the revision as an **edit of one atom** — never as a remove-plus-add pair.

When an atom is added (new `AtomId`) or removed (atom absent) between snapshots, the Delta Stream between those snapshots MUST contain the corresponding event.

## Cross-snapshot reads

There are **no per-call snapshot selectors**: no read or write of snapshot-bound state takes a snapshot parameter, and the calling session's binding is the sole selector ([Session binding](#session-binding)). Two narrow uses of `SnapshotId` arguments remain and are not selectors: snapshot lifecycle methods (`fork`, `switch_snapshot`, `publish_as_canonical`, `discard`) address snapshots by id because snapshots are their subject matter, and a method MAY take a `SnapshotId` as a key into records *about* snapshots (e.g. an archived evaluation run). Neither addresses which snapshot a read or write executes against.

Historical reads are sessions bound to canonical snapshots: to read state as it existed at a published point, a caller binds a session to that canonical snapshot — via `switch_snapshot`, or at session establishment — and issues ordinary reads. The returned atom **MUST** be the atom as it existed in that snapshot: its status, attributes, classification, and other fields reflect the snapshot's state. Write entry points in such a session fail per [Session binding](#session-binding).
