# 10 — Error Model

This chapter defines the binding error model — common error variants, retry semantics, and idempotency guarantees.

## Common error variants

Every container method declares its error variants per the interface file in [`docs/interfaces/`](../interfaces/). The common base variants are:

```typescript
type CommonError =
  | { kind: "NotFound";        entity: string; id: string }
  | { kind: "InvalidInput";    field: string;  reason: string }
  | { kind: "Conflict";        reason: string }                  // logical conflict
  | { kind: "ConcurrentWrite"; reason: string }                  // serialization failure; retryable
  | { kind: "Unauthorized";    action: string }
  | { kind: "Unavailable";     reason: string }                  // transient; retryable
  | { kind: "InternalError";   reason: string }
```

Methods MAY define additional variants. Such variants extend the union; they do NOT replace the common ones.

## Retry semantics

Errors are categorized:

| Category | Variants | Retry guidance |
|---|---|---|
| **Retryable** | `Unavailable`, `ConcurrentWrite`, `Timeout` (LLM Gateway), `RateLimited` (LLM Gateway) | Caller SHOULD retry with backoff. The same operation MAY succeed on retry. |
| **Non-retryable, caller fault** | `InvalidInput`, `NotFound`, `Unauthorized`, `MissingMeta`, `InvalidTransition`, `SchemaViolation` (LLM Gateway) | Caller MUST NOT retry the same operation unchanged. The caller is expected to correct the input or the call. |
| **Non-retryable, deterministic** | `Conflict`, `DuplicateCase` | Retry without changing inputs will produce the same error. |
| **Non-retryable, opaque** | `InternalError`, `ProviderError` | Caller MAY retry with backoff, but the implementation does not guarantee that the retry will succeed. |

Implementations MAY include retry hints in error payloads (e.g., `retry_after_ms`). Callers SHOULD honor hints when present.

## Idempotency contracts

Each interface file's "Idempotency" section declares the idempotency contract per method. The binding rules:

### Atoms

- `propose(fragment, hints?)` MUST be idempotent on `fragment.message_id`. A repeated call within the same snapshot MUST return the same `ProposeResult` (or a semantically equivalent one if the original is no longer cached).
- `update_status(atom_id, outcome, rationale, meta?)` MUST be idempotent on `(atom_id, outcome, meta.idempotency_key?)`. Repeated calls with equivalent arguments are no-ops. Last write wins on `status_changed_at`.

### Ingestion

- `accept(fragment)` MUST be idempotent on `fragment.message_id`.

### Blast Radius

- `analyze(decision_atom_id, options?)` MUST be idempotent on `decision_atom_id` within a snapshot. Re-running produces the same report (modulo bounded non-determinism in the LLM-implicit pass).
- `record_resolution(blast_id, atom_id, resolution, rationale)` MUST be idempotent on `(blast_id, atom_id)`. Last resolution wins.

### Quality

- `record_feedback(call_id, feedback)` MUST be idempotent on `call_id`. Re-recording overwrites prior feedback.
- `add_golden_case(set_id, case)` MUST be idempotent on `case.case_id`.

Methods not listed are NOT idempotent. Callers MUST reason about partial-failure scenarios.

## Atomicity boundaries

A call's atomicity is bounded by the writes it directly issues. Specifically:

- `Atoms.propose(fragment)` is atomic across extraction → conflict classification → status manager mutations → direct-ref cascade. Either all of these complete or none of the changes are reflected in the snapshot.
- `Atoms.update_status(atom_id, outcome)` is atomic against the named atom + any direct-ref cascade it triggers.
- `BlastRadius.analyze(decision_atom_id)` is atomic against the report creation and the bulk `Hanging` cascade — but the LLM-implicit pass MAY be incremental; partial results MAY be visible to other callers via `get_report` while the pass continues.
- `Orchestration.ingest(fragment)` is NOT a single atomic operation — it spans Ingestion, Atoms, Blast Radius (optional), Projection. Atomicity is per-container; the call returns a composite summary, and a partial failure (e.g., Projection re-render fails) MAY leave the snapshot with atoms accepted but projections incomplete. The next `re_render(scope)` call will reconcile.

## Cross-container failures

When a call routes through multiple containers (`Orchestration.ingest`, `Orchestration.query`), a failure in one container MUST NOT silently corrupt another container's state. Specifically:

- If `Ingestion.accept` succeeds but `Atoms.propose` fails, the fragment remains persisted in Ingestion (it's append-only) but no atoms are added. A subsequent retry of `Orchestration.ingest` with the same `message_id` is idempotent — it re-attempts Atoms work without re-ingesting.
- If `Atoms.propose` succeeds but `Projection.re_render` fails, atoms are added but projections are out of date. The delta-stream consumer will catch up; an explicit `re_render(scope)` MAY be invoked to recover.

The implementation MUST NOT roll back successful steps to undo a downstream failure unless explicitly designed to do so. Cross-container compensation is the caller's (or the orchestrator's) concern.

## Error envelope

Implementations MAY use any error transport (exceptions, Result types, error envelopes in responses). The contract is the variant shape, not the transport.

For language-agnostic call patterns, errors MAY be modeled as a discriminated union (TypeScript style) where the success type and the error union are explicit in the return.

## Logging

Errors SHOULD be logged with sufficient context for debugging:

- The method called.
- The relevant identifiers (`atom_id`, `fragment_id`, `blast_id`, `snapshot_id`).
- The error variant + reason.
- The caller context (caller identity, request id, snapshot id).

`InternalError` SHOULD include the underlying stack trace (in implementation-internal logs, not in the error returned to callers).
