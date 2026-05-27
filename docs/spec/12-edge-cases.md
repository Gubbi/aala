# 12 — Edge Cases

This chapter pins down behavior for cases that surface during real use: concurrent operations, malformed inputs, convergence, snapshot transitions, and other corners where the obvious answer isn't.

## Concurrent proposals targeting the same atom

When two `Atoms.propose` calls land at roughly the same time and the Conflict pipeline would classify them as affecting the same canonical atom (e.g., both want to supersede atom `X`):

- The implementation MUST serialize the writes per [04 — Snapshots](./04-snapshots.md). Whichever call wins the serialization runs first; the second call's Conflict pipeline runs against the post-state of the first.
- Both calls MUST succeed (or both MUST fail with `ConcurrentWrite` for the loser if the implementation chose optimistic concurrency).
- The second call's classifications MAY differ from what they would have been pre-first-call (e.g., the new atom from the first call now exists as canonical).
- Neither call MAY observe a partial state from the other.

## Adapt followed by Adapt

A `Hanging` atom is adapted (revised content proposed), reaching `Active`. Before downstream consumers catch up, the atom is again identified as `Hanging` and adapted again. The behavior:

- Each Adapt is a fresh `Atoms.propose` with revised content.
- Each propose generates a new atom (likely classified as `Supersedes` against the prior).
- The atom id of the final `Active` atom is the latest revision's id; earlier revisions are in removal-outcome states (or deleted, per implementation choice).
- Snapshot history retains every revision; the audit trail shows the chain.

## `changes_since(ref)` against a discarded snapshot ref

A consumer holds a `ref` from snapshot `S`. `S` is later discarded. The consumer subsequently calls `changes_since(ref)`. The implementation MUST do one of:

1. Return `Conflict` with reason indicating the snapshot is no longer reachable. The consumer is expected to re-baseline against the current snapshot.
2. Return a re-baseline page: events spanning from the current snapshot's start to its current head, followed by a `next_ref` for ongoing consumption. The page MUST be marked clearly (implementations MAY add a field; the contract is "consumer can detect the re-baseline").

Implementations MUST NOT silently return stale events from the discarded snapshot.

## Snapshot switched mid-operation

`Orchestration.switch_to(new_snapshot)` is called while an in-flight `Orchestration.ingest(fragment)` is running against the previous snapshot:

- The ingest MUST either complete fully against the old snapshot, or fail with `Conflict` indicating the snapshot transitioned mid-call.
- Partial application across snapshots is FORBIDDEN.
- If the ingest succeeds, its results are reachable via reads against the old snapshot id; they MAY or MAY NOT be reachable from the new snapshot depending on whether the new snapshot was forked from the old (post-ingest) or from somewhere else.

## Blast Radius LLM-implicit pass non-convergence

The iterative refinement loop in Blast Radius MUST terminate. Implementations MUST cap iterations (the `BlastOptions.max_iterations`, with an implementation-default cap).

When the cap is reached without reaching a fixed point:

- The report MUST close with the most recent affected set.
- The report MUST record a flag or attribute indicating the cap was hit (so reviewers know the analysis is potentially incomplete).
- Subsequent re-analysis (e.g., `analyze(decision_atom_id)` called again) MAY find a different set.

## Malformed LLM responses

When LLM Gateway invokes a model with a schema and the response fails validation after all configured retries:

- The Gateway raises `SchemaViolation` per [09](./09-llm-gateway.md).
- The calling container (Atoms extraction, Conflict judge, etc.) MUST treat this as a hard failure for the call.
- For `propose`: the fragment was extracted but classification failed. The implementation MAY:
  - Reject the fragment outright (fragment marked, atoms not added).
  - Add atoms with the extractor's output but no Conflict classification (atoms marked with `Unclear` outcome or similar).
- Either is conformant; the choice is documented per implementation.

## Reference to a non-existent atom

An incoming fragment produces an atom with `references` or `depends_on` pointing at an atom id that doesn't exist in the snapshot. The implementation MUST:

1. Not raise an error solely on that ground. Dead references are valid until the referenced atom is added.
2. Optionally surface a warning via an implementation-specific mechanism (e.g., attribute, event).
3. Permit the atom to enter the snapshot.

If the referenced atom is later added (e.g., from a subsequent fragment), no further action is required — the reference resolves.

If the referenced atom is in a removal outcome (deleted or tombstoned), the reference is a dead ref. Implementations MAY:
- Surface a warning at read time.
- Periodically scan for dead refs and emit events (the Integrity Checker in Hierarchical Navigation is one mechanism, but Atoms itself is not required to do this).

## Definition atom with circular alias chain

A `DefinitionAtom` declares `aliases = ["X"]`. A second `DefinitionAtom` with `subject = "X"` declares `aliases = [<first atom's subject>]`. This is a circular alias chain.

The Conflict pipeline MUST detect and break circularity. The implementation MAY:
- Reject the second proposal as `Duplicate`.
- Use `canonical_for` to deterministically pick one as canonical and the other as an alias-only reference.
- Raise a warning and let human review resolve.

The specific resolution is implementation choice. The contract is "do not loop indefinitely; do not crash."

## Empty fragment

A `NormalizedFragment` with an empty `payload`:

- The Relevance Filter MUST be permitted to reject it.
- If accepted, the Extractor MAY produce zero atoms (no `Added` events emitted).
- The fragment MUST still be persisted in Raw Store for provenance.

## Fragment with a duplicate `message_id`

Per the idempotency rule, `Ingestion.accept` and `Atoms.propose` are idempotent on `fragment.message_id`. A second call with the same `message_id`:

- MUST NOT add the fragment a second time.
- MUST return the result of the original call (or a semantically equivalent one).
- MUST NOT re-trigger extraction or Conflict classification.

If an implementation discards its idempotency-tracking state (e.g., across a restart), it MAY re-trigger work. Callers MUST be prepared for this — the underlying contract is "the resulting state is consistent with a single application of the fragment," not "the API call is observed exactly once."

## Concurrent `update_status` on the same atom

Two simultaneous `Atoms.update_status(atom_id, ...)` calls on the same atom:

- MUST be serialized.
- Each MUST individually validate against the current state (last-state wins, after serialization).
- If the first transitions `Active → Hanging` and the second transitions `Active → Deprecated`, only the first's transition is valid against `Active`; the second's pre-condition (`Active`) is no longer met when it runs.
- The second MUST then either be rejected as `InvalidTransition` (if the new pre-condition fails) or proceed if the new pre-condition still permits (e.g., `Hanging → Deprecated` is valid).
- The implementation MAY use optimistic concurrency (return `ConcurrentWrite` to the loser, expect retry) or pessimistic (block the loser, run sequentially).

## Snapshot publication race

Two callers simultaneously call `Orchestration.publish_as_canonical(snapshot_id)` on the same snapshot:

- The publication MUST be atomic; both callers MUST observe a single successful publication (one returns success, the other MAY return `AlreadyCanonical` or also success — the underlying state is identical).
- The state MUST NOT be observable as "publishing in progress" by other callers; the snapshot is either `working` or `canonical`.

## Implementation extensions

Implementations MAY support additional behaviors beyond what this spec requires (caching modes, observability sinks, additional atom types, etc.). Such extensions:

- MUST NOT change the behavior of any case described above.
- MUST be discoverable via `capabilities()` extension fields or other documented mechanisms.
- MUST be opt-in for callers; default behavior MUST match the spec.
