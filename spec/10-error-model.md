# 10 — Error Model

This chapter defines the binding error model: a three-tier error chain, the classification that drives caller behavior, the mandated observability emission, and the idempotency / atomicity / cross-container guarantees that surround failures.

The shapes referenced here (`FixDomain`, `Recoverability`, `Severity`, `PrimitiveError`, `ActivityError`, `OperationError`, `Activity`, `OperationKind`) are defined in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md#errors--the-three-tier-model).

## The three-tier chain

aala is designed to be composed from containers that may be built by different vendors and wired together by an integrator. The error model is the contract across that boundary. Every error is a chain of up to three tiers, each with a distinct **audience** and **owner**:

| Tier | Name | What it is, in aala terms | Audience | Owned by | Standardized? |
|---|---|---|---|---|---|
| 1 | **Operation Error** | the whole job the caller asked for — the `correlation_id`'d operation | end users | customer touch points (product / integration surface) | shape standardized; present only at the edge |
| 2 | **Activity Error** | a unit of work *within* the operation — "what the system was doing for you, and that it didn't work" | integrators + end users | every container / component | **fully standardized — closed `Activity` vocabulary + classification** |
| 3 | **Primitive Error** | the atomic technical leaf failure + diagnostics | developers | the container vendor, privately | shape standardized; `type` values **not** spec-enumerated |

This maps exactly onto the operation vocabulary in the rest of the spec: **Operation = the `correlation_id` scope** (`ingest:` / `request:` / `system:`), **Activity = a work-unit inside it**, **Primitive = the leaf**.

The system proper deals in **Activity + Primitive**. Tier 1 is the edge framing a customer touch point applies; the internals are not required to produce it.

### Tier roles

- **Operation Error (Tier 1).** Frames the failure in client intent ("couldn't ingest your document"). It **delegates** all classification to the Activity it wraps — it declares no classification of its own. Owned by the touch point, which maps it to its product vocabulary.
- **Activity Error (Tier 2).** Names the user-meaningful unit of work that failed (the `activity`) and says, in plain terms, *what happened*. It is **not** a dump of the technical component hierarchy — `component` is a secondary field for "where to look," not the headline. It is the single source of truth for **classification** (`fix_domain` + `recoverability` → derived `severity`). Every Activity Error **MUST** wrap a Primitive Error as its `cause`.
- **Primitive Error (Tier 3).** The concrete leaf failure with technical diagnostics. Implementation-specific and reserved for the technical audience: its `type` and `details` are **not** part of the cross-vendor contract, are surfaced to telemetry and error logs, and are **filtered out of any end-user-facing render**.

**Cross-container propagation.** When one container's activity fails because a downstream container's activity failed, the downstream Activity Error propagates to the caller **unchanged** — the calling container does not re-wrap it in its own activity (e.g. `BlastRadius.record_resolution` surfaces Atoms' `transitioning` / `invalid_transition` as-is). Only a customer touch point adds a tier, by framing the Operation Error.

## Classification

Classification is declared once, at the Activity tier, and propagates upward to the Operation tier unchanged.

### FixDomain — where the fix lives

| Value | Meaning |
|---|---|
| `client` | Fix is in the caller's input or usage. |
| `data` | Fix requires repairing stored content (corrupt/malformed atom, snapshot, index). |
| `infra` | Fix is in the underlying infrastructure (permissions, disk, network, LLM provider). |
| `aala` | Fix is in an aala container's code (an invariant was violated). |

### Recoverability — what the caller should do

| Value | Meaning |
|---|---|
| `retryable` | Transient; retry the same call after backoff. |
| `user_action` | Caller must change input or usage, then retry. |
| `operator_action` | Operator must repair infrastructure or data, then retry. |
| `not_recoverable` | Invariant violated; escalate to a developer. |

### Severity — derived, never declared

Severity is a deterministic function of the **pair** `(fix_domain, recoverability)`, so a declared severity can never drift from the actual semantics. Keying on both axes lets the meaning of a recoverability differ by where the fix lives — e.g. a `not_recoverable` that is the *client's* misuse (asking for an absent capability) is benign (`warn`), while a `not_recoverable` invariant violation in aala's own code is an `error`.

| `fix_domain` ↓ \ `recoverability` → | `retryable` | `user_action` | `operator_action` | `not_recoverable` |
|---|---|---|---|---|
| `client` | warn | warn | warn | **warn** (misuse — e.g. `not_wired`; nothing is broken) |
| `data` | warn | warn | error | error |
| `infra` | warn | warn | error | error |
| `aala` | warn | warn | error | error (invariant violated — a bug) |

In words: `retryable` and `user_action` are always `warn` (transient, or the caller's to fix); `operator_action` and `not_recoverable` are `error` **except** when `fix_domain = client`, where they remain `warn` because the situation is caller misuse, not a system fault. The full pair is retained as the derivation surface so future cases can refine a cell without changing the signature.

An implementation **MUST NOT** carry a `severity` that disagrees with this table.

### Callers branch on classification, not on names

The control-flow decision (*retry / surface / alert / abort*) is driven entirely by `recoverability`:

| `recoverability` | Caller behavior |
|---|---|
| `retryable` | Retry with backoff (honor `retry_after_ms` if present). |
| `user_action` | Surface to the caller; do not retry unchanged. |
| `operator_action` | Alert an operator; do not retry unchanged. |
| `not_recoverable` | Escalate; treat as a bug. |

Callers **MUST NOT** match on any tier-3 `type` — the leaf is diagnosis, not contract. Where behavior genuinely differs by case beyond classification, callers discriminate on the **`(activity, end_state)` pair** — both closed, registered vocabularies (see [Activity end-states](#activity-end-states)). `activity`, `message`, and `hint` are for human communication; the `cause` Primitive is for the technical audience.

## The standardized Tier-2 vocabulary

The closed set of activities every container honors is enumerated in [Appendix A § Activities (Tier 2)](./appendix-a-registries.md#activities-tier-2). Each names a unit of work the system performs on the caller's behalf; a method's **Errors** declaration lists the `(activity, end_state)` pairs it can raise.

### The standardized Tier-1 vocabulary

The closed set of operations a customer touch point frames is enumerated in [Appendix A § Operations (Tier 1)](./appendix-a-registries.md#operations-tier-1). Each corresponds to a `correlation_id` scope.

This vocabulary does double duty as the **access-control vocabulary**: every interface method declares exactly one operation type in its **Access** line, and the perimeter checks the calling session's grants against it ([02 — Conformance § Access control](./02-conformance.md#access-control)).

## Activity end-states

An Activity Error names more than a classification — it names **which error end-state the activity terminated in**. `end_state` is a required field, and the set of error end-states is **closed per activity**, registered like the spec's other closed sets in [Appendix A § Activity end-states](./appendix-a-registries.md#activity-end-states). Classification is fixed **per `(activity, end_state)` pair** by that registry: an implementation MUST emit the registered `fix_domain` / `recoverability` for the pair (with `severity` derived per the table above) and MUST NOT invent end-states. Extending an activity's end-state set is a spec change ([11 — Versioning](./11-versioning.md)).

Success is not represented in the registry: an activity that completes returns a result, and conflict-pipeline outcomes are results too (see the fence below). The registry holds *error* end-states only.

### Universal end-states

Every activity's end-state set includes the five universal end-states enumerated in [Appendix A § Universal end-states](./appendix-a-registries.md#universal-end-states). Because they apply to every activity, a method's **Errors** declaration does not repeat them — it lists only its per-activity pairs.

### Per-activity end-states

The per-activity pairs — each with its registered `fix_domain` / `recoverability` — are enumerated in [Appendix A § Per-activity end-states](./appendix-a-registries.md#per-activity-end-states).

Notes:

- An absent optional capability surfaces as the universal `not_wired` — e.g. Blast Radius not deployed → `analyzing_impact` / `not_wired`; a benchmark never configured → `evaluating` / `not_wired`.
- A method's **Errors** declaration in [`interfaces/`](../interfaces/) lists the `(activity, end_state)` pairs it can raise — that list is the per-method contract.
- **Session-binding failures are blanket pairs.** A write entry point invoked in a session bound to a non-`working` snapshot fails with `snapshotting` / `canonical_protected`; any call in a session whose bound snapshot has been discarded fails with `snapshotting` / `discarded` ([04 — Snapshots § Session binding](./04-snapshots.md#session-binding)). Like the universal end-states, these two pairs apply across the surface and are not repeated in per-method **Errors** declarations.
- **Access-control failures are blanket pairs.** Any call can fail at the perimeter with `authorizing` / `unauthenticated` (no session established) or `authorizing` / `denied` (missing grant), per the check in [02 — Conformance § Access control](./02-conformance.md#access-control). Like the universal end-states, these two pairs apply across the surface and are not repeated in per-method **Errors** declarations; the per-method **Access** line declares the operation type the check uses.

## Composition and observability are separate concerns

Two orthogonal contracts apply to the same chain.

- **Composition (structural).** Classification propagates up the `cause` chain: the Activity declares it; the Operation delegates. The chain is traversable end-to-end, and the leaf's typed fields remain reachable at the surface — no information is lost between tiers.
- **Observability (emission).** A single emission at the surface walks the `cause` chain and contributes each tier's fields into one telemetry event. Emission never changes the structural contract.

## Mandated observability

aala **mandates [OpenTelemetry](https://opentelemetry.io/) emission** as the observability baseline. Because containers may be independently built and composed, a fixed emission contract is what guarantees a coherent end-to-end trace across vendor boundaries. OTel is an interop standard (alongside the [wire profiles](./13-wire-profiles.md)), not a swappable backing service — it is part of the contract, not deferred to a deployment ADR.

### Who emits, and where

Every error is emitted **exactly once, at construction, by the owner of the tier that constructed it**:

| Error | Emitted by | Emitted onto |
|---|---|---|
| Activity Error (carrying its full Primitive `cause`) | the container that raised it | the container's own active span, at its API boundary |
| Operation Error | the customer touch point that framed it | the operation-level (root) span the touch point owns |

A container that *propagates* a downstream Activity Error unchanged MUST NOT re-emit it — it marks its own span as errored and lets the original emission stand. Operation Errors exist only at customer touch points, so Tier-1 emission happens only there; container internals never emit Tier 1.

### Endpoint and configuration

aala defines no observability configuration surface of its own. As is normal practice when a specification incorporates a standard by reference, the standard's own configuration mechanism governs: implementations MUST honor the OpenTelemetry SDK's standard configuration surface — the `OTEL_*` environment variables defined by the OTel specification (e.g. `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_PROTOCOL`, `OTEL_SERVICE_NAME`, `OTEL_RESOURCE_ATTRIBUTES`) — and MUST NOT invent parallel knobs for the same settings. The mandated baseline on top of that:

- **OTLP export** from every container.
- **Resource identity**: `service.name` names the container (e.g. `aala-atoms`); `service.version` carries its interface version.
- **Context propagation**: W3C Trace Context across every container boundary, on every wire profile.
- **Correlation linkage**: every span carries the operation's `correlation_id` as the `aala.correlation_id` span attribute, so traces join with the contract thread (next section).

### `trace_id` — the operation's `correlation_id`, carried on the error

The `trace_id` field on Activity and Operation Errors **is** the operation's `correlation_id` (`ingest:` / `request:` / `system:`):

- As a **contract field**, it is returned to the caller in the error and written to every error log line — the explicit key joining the returned error, the logs, and the trace.
- In telemetry it is carried as the **`aala.correlation_id` attribute** on every span and emitted event. It is **not** a W3C Trace Context trace-id — trace identity is a 128-bit identifier the OTel SDK mints and propagates — and an implementation MUST NOT use the `correlation_id` as, or derive from it, a trace-id. The two correlate by attribute: filtering on `aala.correlation_id` joins one operation's spans across containers, on whatever traces they belong to.
- OTel spans themselves remain pure observability artifacts — the runtime never branches on a span. The contract element is the field; the span correlates *to* it.

### Emission cascade and field naming

One `emit()` at the surface cascades down the `cause` chain; each tier contributes its fields to a single event:

```
emit()                          → operation-tier fields (aala.operation, aala.correlation_id)
  → cause (Activity)            → error.type (the pair), aala.activity, aala.end_state, aala.component,
                                   aala.hint, aala.fix_domain, aala.recoverability, aala.severity
       → cause (Primitive)      → exception.type, exception.message, code.file.path / code.line.number,
                                   and typed detail fields under aala.error.<type>.<field>
```

Attribute naming is fixed — there is no free-form naming. Where the [OTel semantic conventions](https://opentelemetry.io/docs/specs/semconv/) define a standard attribute, emission MUST use it; implementations MUST target semantic conventions **v1.29.0 or later** (the attribute names below are from that line). Every aala-specific field is emitted under the **`aala.` namespace** — the semconv-owned namespaces (`error.`, `exception.`, `code.`) are reserved by OTel and MUST NOT carry custom attributes:

| Field | Attribute |
|---|---|
| Error class | `error.type` — the OTel-standard low-cardinality classifier; set to the `(activity, end_state)` pair as `<activity>/<end_state>` (e.g. `transitioning/invalid_transition`) |
| Leaf type | `exception.type` — the Primitive's `type` |
| Leaf message | `exception.message` — the Primitive's `message` |
| Leaf location | `code.file.path` / `code.line.number` (or domain location) — from `location` |
| Correlation linkage | `aala.correlation_id` — the operation's `correlation_id` (the error's `trace_id` field) |
| Operation kind | `aala.operation` — the Tier-1 `operation`, when an Operation Error is present |
| Shared semantic fields | `aala.activity`, `aala.end_state`, `aala.component`, `aala.hint`, `aala.fix_domain`, `aala.recoverability`, `aala.severity` |
| Tier-specific leaf fields | `aala.error.<type>.<field>` — namespaced by the Primitive's `type` so tiers never collide |

The Primitive's technical detail is emitted to telemetry and error logs in full; it is **filtered out of any end-user-facing render** (only the Operation/Activity `message` + `hint` reach the end user).

## Conflict-pipeline outcomes are results, not errors

A hard-block or soft-prompt conflict outcome ([06](./06-conflict-classification.md)) is **not an Activity Error** — it is a typed *result* of `Atoms.propose`. The proposal was processed successfully; the outcome describes a situation needing auto / soft / hard resolution. Conflict outcomes keep their own resolution taxonomy and **do not** carry `fix_domain` / `recoverability`. Errors mean the call itself failed; conflict outcomes mean the call succeeded and the proposal needs resolution. This fence is binding: an implementation **MUST NOT** surface a conflict outcome as an Activity Error, nor an Activity Error as a conflict outcome.

*Which* validations sit on which side — the **schema-violation fence** — is drawn once, in [06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes). The registry rows above and the interface **Errors** declarations reference that boundary; they do not redraw it.

## Idempotency contracts

The vocabulary — what **idempotent** and **last-write-wins** mean, and that they are distinct guarantees — is defined in [01 — Conventions § Idempotency and last-write-wins](./01-conventions.md#idempotency-and-last-write-wins). The per-method contracts are declared once, in the idempotency registry in [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency) — this chapter never restates them. It adds only the rules where idempotency meets the error model:

- **A replay is a success, not an error.** A call a method recognizes as a replay MUST succeed with the recorded (or semantically equivalent) result. An implementation MUST NOT surface a replay as an Activity Error.
- **A failed atomic call records nothing to replay.** For a method whose atomicity boundary covers the whole call ([§ Atomicity boundaries](#atomicity-boundaries)), an Activity Error means no recorded request exists under its idempotency key; the retry executes as a fresh attempt.
- **Retry-safety by composition.** `Orchestration.ingest` is not atomic; its retry-safety is composed — a retry replays completed per-container steps as no-ops and resumes incomplete ones ([§ Cross-container failures](#cross-container-failures)).
- **Everything else.** A mutating method with no registry row carries no idempotency contract: a repeated call is a new call, and callers MUST reason about partial-failure scenarios before retrying.

## Atomicity boundaries

A call's atomicity is bounded by the writes it directly issues:

- `Atoms.propose(fragment)` is atomic across extraction → conflict classification → status manager mutations → cascade fixpoint (per [07 — Atom Lifecycle](./07-atom-lifecycle.md)). Either all complete or none of the changes are reflected in the snapshot.
- `Atoms.update_status(atom_id, outcome)` is atomic against the named atom + the full cascade fixpoint it triggers across all three channels (predicate-kind, scope-premise, derivation invalidation).
- `Atoms.update(atom_id, changes, rationale)` is atomic across validation → content mutation → the entailment recompute of affected derivations. Either all complete or the snapshot is unchanged.
- `BlastRadius.analyze(input)` is read-only and atomic against report creation — but the LLM-implicit pass MAY be incremental; partial results MAY be visible via `get_report` while the pass continues.
- `Orchestration.ingest(fragment)` is NOT a single atomic operation — it spans Ingestion, Atoms, and Blast Radius (optional). Atomicity is per-container; the call returns a composite summary, and a partial failure MAY leave the snapshot with atoms accepted but other container state incomplete. The Projection facet derives from container content, so its read views reconcile as the host container's content settles.

## Cross-container failures

When a call routes through multiple containers (notably `Orchestration.ingest`; a dispatched read executes on a single container — [`interfaces/orchestration.md § query`](../interfaces/orchestration.md#query)), a failure in one container MUST NOT silently corrupt another's state:

- If `Ingestion.accept` succeeds but `Atoms.propose` fails, the fragment remains persisted in Ingestion (append-only) but no atoms are added. A subsequent retry of `Orchestration.ingest` with the same `message_id` is idempotent — it re-attempts Atoms work without re-ingesting.
- The Projection facet is a read-only view over a container's own content; it has no write step that can fail mid-ingest. If atoms are added, the facet reflects them on the next read (or via its `DocChanged` / `IndexUpdated` events), and Delta-Stream consumers catch up from the host container's stream.

The implementation MUST NOT roll back successful steps to undo a downstream failure unless explicitly designed to do so. Cross-container compensation is the caller's (or the orchestrator's) concern. When Tier 1 is present, the composing touch point wraps the failing container's Activity Error in an Operation Error and delegates its classification.

## Logging

Every error MUST be logged with enough context to diagnose without reading source:

- The method called, the `activity`, and the `end_state`.
- The relevant identifiers (`atom_id`, `fragment_id`, `blast_id`, `snapshot_id`).
- The `trace_id` (= `correlation_id`).
- The classification triple (`fix_domain`, `recoverability`, derived `severity`).
- The full `cause` Primitive — `type`, `message`, `location`, and `details` — which is logged even though it is withheld from end-user-facing renders.
