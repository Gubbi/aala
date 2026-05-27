# Shared Types

Types and conventions referenced from multiple container interfaces. Read this before the per-container files.

## Identifiers

All identifiers are opaque strings unless otherwise specified. Their internal structure (UUID, content-hash, path-derived, etc.) is implementation-specific and not part of this contract. Identifiers are stable: a given id refers to the same logical entity across reads, even across snapshots.

```typescript
type AtomId        = string  // globally unique atom identifier
type FragmentId    = string  // raw fragment in Ingestion
type SnapshotId    = string  // snapshot identifier
type Ref           = string  // opaque checkpoint for changes_since
type IngestId      = string  // handle returned by Atoms.propose
type BlastId       = string  // blast report id
type SectionPath   = string  // projection section path (e.g., "domains/payments/services/checkout.md#flow")
type ScopePath     = string  // scope path (e.g., "domains/payments/services")
type ContentKind   = string  // e.g., "slack.message", "meeting.transcript", "doc.adr"
type UseCaseKey    = string  // registered LLM Gateway use case key (e.g., "aala.extraction")
```

## NormalizedFragment

The shape Ingestion produces and Atoms consumes.

```typescript
interface NormalizedFragment {
  message_id:    string                  // stable, source-supplied id (idempotency key)
  sender:        string
  recipients:    string[]
  payload:       string                  // the prose / message body
  timestamp:     string                  // ISO 8601
  content_kind:  ContentKind
  source_uri:    string                  // back-reference to source artifact
  extras:        Record<string, unknown> // source-specific structure
}
```

## Atom (data model)

The conceptual atom shape. Detailed semantics, status transitions, and removal outcomes are specified in [`atoms.md`](./atoms.md).

```typescript
type AtomStatus =
  | "active"
  | "hanging"
  | "deferred"
  | "deprecated"
  | "superseded"        // present only in tombstone-style implementations
  | "duplicate"
  | "rejected"
  | "removed"

type AtomType =
  | "decision"
  | "definition"
  | "dependency"
  | "constraint"
  | "behavior"
  | "capability"
  | "scenario"

interface Atom {
  id:                   AtomId
  type:                 AtomType
  subject:              string
  predicate:            string
  object:               string
  attributes:           Record<string, unknown>
  status:               AtomStatus
  status_reason?:       string
  status_changed_at?:   string  // ISO 8601
  premises:             Premise[]
  sources:              Source[]
  confidence:           number  // 0.0–1.0
  confidence_factors:   Record<string, number>
  scope:                Scope
  references:           AtomId[]  // outgoing
  depends_on:           AtomId[]  // outgoing dependency edges
}

interface DecisionAtom extends Atom {
  type:                 "decision"
  rationale:            string
  supersedes:           AtomId[]
  decision_metadata:    Record<string, unknown>
}

interface DefinitionAtom extends Atom {
  type:                 "definition"
  aliases:              string[]
  see_also:             AtomId[]
  canonical_for?:       string
}

interface Source {
  uri:                  string
  quote:                string
  extracted_at:         string  // ISO 8601
}

interface Premise {
  key:                  string
  value:                string
}

interface Scope {
  environment?:         string
  time_scope?:          TimeScope
  conditions:           string[]
}

interface TimeScope {
  effective_from?:      string  // ISO 8601
  effective_to?:        string  // ISO 8601
}
```

## Change-stream events

Every stateful container exposes its own change stream via `changes_since(ref)`. The event type is union-typed per container; specific variants are listed in each container's interface file.

The shared envelope:

```typescript
interface ChangeEvent<TVariant> {
  ref:           Ref           // monotonic checkpoint after applying this event
  kind:          string        // event kind, e.g., "Added", "StatusChanged"
  payload:       TVariant      // kind-specific payload
  emitted_at:    string        // ISO 8601
  source_ref?:   Ref           // origin checkpoint (when this event was caused by an upstream delta)
}

interface ChangesPage<TVariant> {
  events:        ChangeEvent<TVariant>[]
  next_ref?:     Ref           // if absent, caller is caught up
}

// Shared method signature on every stateful container
function changes_since(ref: Ref | null, limit?: number): Promise<ChangesPage<unknown>>
```

`ref = null` returns events from the beginning of the current snapshot. The page size cap is impl-specific; callers must keep paging on `next_ref`.

## SnapshotRef

The handle the Orchestration container exposes for snapshots; consumed by every other container.

```typescript
interface SnapshotRef {
  id:            SnapshotId
  parent?:       SnapshotId   // forked from this snapshot
  state:         "working" | "canonical" | "discarded"
  created_at:    string
}
```

## Common error variants

Container methods declare a union of error variants per call. The common ones:

```typescript
type CommonError =
  | { kind: "NotFound"; entity: string; id: string }
  | { kind: "InvalidInput"; field: string; reason: string }
  | { kind: "Conflict"; reason: string }                // logical conflict (e.g., a stale ref)
  | { kind: "ConcurrentWrite"; reason: string }         // serialization failure; safe to retry
  | { kind: "Unauthorized"; action: string }
  | { kind: "Unavailable"; reason: string }             // transient
  | { kind: "InternalError"; reason: string }
```

Per-method error variants extend this. Callers should always handle `Unavailable` and `ConcurrentWrite` as retryable.

## Idempotency

A method is **idempotent on key K** when repeated calls with the same K-derived inputs produce the same effect as a single call (modulo the response body, which may differ).

| Pattern | Idempotency key |
|---|---|
| `Ingestion.accept(fragment)` | `fragment.message_id` |
| `Atoms.propose(fragment, …)` | `fragment.message_id` |
| `Atoms.update_status(atom_id, outcome, rationale, meta?)` | `(atom_id, outcome, meta.idempotency_key?)` — the last write wins by `status_changed_at` |
| `BlastRadius.record_resolution(blast_id, atom_id, resolution, …)` | `(blast_id, atom_id)` — last resolution wins |

Methods not listed as idempotent are not — callers must reason about partial-failure scenarios.

## Concurrency

Unless otherwise stated:

- **Reads** against a snapshot are concurrent-safe and serializable with each other.
- **Writes** against a snapshot are serialized per-snapshot. Concurrent writes to the same snapshot are either queued (blocking) or fail with `ConcurrentWrite` (caller retries). The choice is per-implementation; the contract is "no partial / interleaved writes."
- **Cross-snapshot operations** (forking, switching) are atomic at the snapshot level — observers see either the pre or the post state of a snapshot lifecycle event, never an intermediate.

## Versioning

Each container interface declares a `version` string at the top of its file (e.g., `aala-v1`). The version covers method signatures + error variants + event kinds. A **patch-compatible** change adds optional fields or new variants without removing or repurposing existing ones; a **breaking** change does not.
