# 04 — Snapshots

A **snapshot** is the unit of coherence across containers. This chapter defines its lifecycle and the invariants every conformant implementation must honor.

## Definition

A snapshot is an addressable, coherent version of all snapshot-bound state — atoms, projections, blast reports, navigation trees, and any other state a container persists. A snapshot is identified by an opaque `SnapshotId`.

The realization is implementation-specific: a snapshot MAY be a git ref + working tree, a row in a versioning table, a content-addressed manifest, an immutable in-memory structure, or anything else with the right semantics. The contract is the behavior described in this chapter, not the realization.

## States

Every snapshot is in exactly one of three states:

| State | Meaning |
|---|---|
| `working` | Mutable. Capability containers MAY write to it. At most one snapshot at a time MAY be the **current** working snapshot (see Concurrency below). |
| `canonical` | Immutable. Represents committed state. Reads against canonical snapshots MUST be stable and reproducible. |
| `discarded` | Working snapshots not promoted to canonical. Eligible for garbage collection at the implementation's discretion. |

## Transitions

Allowed transitions:

- (none) → `working` — via `Orchestration.fork()`.
- `working` → `canonical` — via `Orchestration.publish_as_canonical(id)`.
- `working` → `discarded` — via `Orchestration.discard(id)`.
- `canonical` → (none) — canonical snapshots MUST NOT be discarded by the implementation. They MAY be archived per implementation policy, but their identity and reachability MUST be preserved for as long as referenced.
- `discarded` → (none) — implementations MAY garbage-collect discarded snapshots.

`canonical → working` is **NOT** a valid transition. To work from a canonical snapshot, a caller forks a new working snapshot from it.

## The "current" snapshot

At any moment, every capability container MUST operate against the same snapshot. This is the **current working snapshot**, returned by `Orchestration.current_snapshot()`.

A capability container reading or writing state MUST do so against the current snapshot unless explicitly given a different `SnapshotId`. Some interface methods accept an explicit snapshot parameter for historical reads; those reads MUST be served from the named snapshot.

## Coherence invariant

Cross-container reads at a single snapshot identifier MUST be coherent. Specifically:

> For any two reads `r1` and `r2` issued against the same `SnapshotId`, the state observed by `r1` and `r2` MUST be consistent with a single point in the snapshot's history. No interleaving with concurrent writes against that snapshot MAY produce a state that is not reachable by sequencing the writes.

Implementations achieve this via any of:

- Single-writer-per-snapshot with snapshot-isolated reads.
- Multi-version concurrency control (MVCC) with a stable read transaction.
- Append-only logging with derived projections built deterministically from the log.

The contract is the property, not the technique.

## Lineage

A snapshot MAY have a parent (the snapshot it was forked from). Lineage MUST be preserved for as long as either the snapshot or any of its descendants exist.

`SnapshotRef.parent` returns the parent's `SnapshotId` (or absent for root snapshots).

`Orchestration.list_snapshots({ parent_of: id })` MUST return all snapshots forked from `id`.

## Canonicalization

Publishing a working snapshot as canonical happens by one of two mechanisms:

1. **aala-managed:** `Orchestration.publish_as_canonical(id)` actively promotes the snapshot. The implementation transitions the state from `working` to `canonical` and emits the corresponding `SnapshotPublished` event.

2. **Externally-managed:** the publishing happens outside aala's API (e.g., the user committing via `git`). In this case, `publish_as_canonical` MAY be a no-op or a detection trigger; the implementation observes the external state change and emits `SnapshotPublished` accordingly.

Both are conformant. The published `Orchestration.capabilities()` declaration SHOULD indicate which mechanism the implementation uses.

After publication, the snapshot's state MUST be `canonical` from all containers' perspectives, and reads of that snapshot MUST be stable.

## Switching

`Orchestration.switch_to(snapshot_id)` MUST atomically update which snapshot is current. Capability containers' `changes_since(ref)` consumers MUST re-baseline against the new snapshot's stream. Implementations MAY do this by:

- Tracking a per-(container, snapshot) consumer checkpoint and switching the active one.
- Re-issuing each consumer's `changes_since` with a new starting ref.

In-flight cross-container operations at the moment of switch MUST either complete against the old snapshot or fail with `Conflict` so the caller can retry — partial execution across snapshots is NOT permitted.

## Removal outcomes and snapshots

When an atom transitions to a removal outcome (`Superseded` / `Duplicate` / `Rejected` / `Removed`):

- The atom MAY be deleted from the current snapshot (it does not appear in subsequent reads of that snapshot).
- The atom MAY be retained as a tombstone with the removal-outcome status.

Either is conformant. Crucially: the atom MUST remain reachable in the **parent** snapshot at the time of the transition. Snapshot history is the audit trail; removing an atom from snapshot `S` MUST NOT affect its presence in any ancestor snapshot.

## Atom identity across snapshots

An atom's `AtomId` is stable across snapshots. The same logical atom MUST keep the same `id` through any `Updated` event and across any number of snapshot transitions. An atom that was in snapshot `S` and is in snapshot `S+1` MUST have the same `id` in both.

When an atom is added (new `AtomId`) or removed (atom absent) between snapshots, the delta stream between those snapshots MUST contain the corresponding event.

## Cross-snapshot reads

`Atoms.get_by_id(atom_id)` and similar reads operate against the current snapshot by default. Implementations MAY expose an explicit snapshot parameter to enable historical reads. The returned atom MUST be the atom as it existed in that snapshot — its status, attributes, and other fields reflect the snapshot's state.
