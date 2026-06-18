# 07 — Atom Lifecycle

Atoms move through a defined state machine. This chapter specifies the states, the legal transitions, the API through which transitions happen, and the cascade rules that propagate status changes through the snapshot.

## States

### Present states

| State | Meaning |
|---|---|
| `active` | Currently held. The default for new atoms. |
| `hanging` | Premise may have changed; awaiting resolution. The atom is no longer asserted with confidence but has not been withdrawn. |
| `deferred` | Known to be hanging; the team chose to defer resolution. Stable until explicitly revisited. |
| `deprecated` | Still holds for existing dependents; no new dependencies SHOULD be added. Stable. |

### Removal outcomes

| Outcome | Meaning | Required meta |
|---|---|---|
| `superseded` | Claim succession — fully absorbed by a compatible refinement (gated; see [Revision, succession, and the prior atom's status](#revision-succession-and-the-prior-atoms-status)) | `meta.superseded_by` REQUIRED |
| `duplicate` | Was a duplicate of a canonical atom | `meta.duplicate_of` REQUIRED |
| `rejected` | Proposal rejected; never made it to canonical | — |
| `removed` | Generic removal | — |

Applying a removal outcome to an atom that **was committed** to the snapshot is a **two-step process**: the atom MUST first transition to the removal-outcome state, that transition MUST be committed — the atom then remains as a **tombstone** carrying that state; it is never deleted directly — and physical deletion happens only via the optional [garbage-collection step](#garbage-collection) below.

An atom that was **never committed** is not tombstoned. A proposal rejected by the Conflict pipeline (`rejected`, or any hard-block outcome) and a `duplicate` / conflict detected before the atom entered the snapshot are simply **not added** — there is nothing to tombstone, and such outcomes MAY be auto-resolved. The non-entry is still recorded as a `Rejected` audit event carrying the candidate ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)), but no tombstone atom persists.

This guarantees every *committed* atom's removal is a recorded, auditable transition rather than a silent disappearance, while never-committed proposals leave no tombstone clutter.

## Transition table

Legal transitions from each source state:

| From → To | active | hanging | deferred | deprecated | superseded | duplicate | rejected | removed |
|---|---|---|---|---|---|---|---|---|
| `active` | — | ✓ | ✗ | ✓ | ✓ | ✓ | ✗ | ✓ |
| `hanging` | ✓ | — | ✓ | ✓ | ✓ | ✓ | ✗ | ✓ |
| `deferred` | ✓ | ✓ | — | ✓ | ✓ | ✓ | ✗ | ✓ |
| `deprecated` | ✗ | ✓ | ✗ | — | ✓ | ✓ | ✗ | ✓ |
| (proposal time only) | ✓ (initial) | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ | ✗ |

Rules:

- `active` is the initial state for any successfully proposed atom.
- `rejected` is reachable only from the proposal stage — once an atom is canonical, it cannot be rejected (use `removed` or `superseded` instead).
- `deprecated` MUST NOT transition back to `active` directly; it MAY first transition to `hanging` (e.g., on revision attempt) and then to `active`.
- Removal outcomes are terminal — once an atom is in a removal outcome, no further transitions are allowed except correction (handled as a new atom proposal, not a state change).

Any transition not in the table is illegal and MUST fail with a `transitioning` / `invalid_transition` Activity Error (see [10 — Error Model](./10-error-model.md#activity-end-states)). The diagonal (`from == to`) is not a legal transition, but a repeated call naming the atom's current state is handled by the idempotency check, which **precedes** the legality check (step 1 below) — such a call succeeds as a no-op rather than failing as illegal.

## API for transitions

`Atoms.update_status(atom_id, outcome, rationale, meta?)` is the only API surface for explicit transitions. The implementation MUST:

1. Check idempotency **before** legality: when `outcome` equals the atom's current `status.state` and the arguments are equivalent under the method's idempotency key (registry row in [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency)), the call is a successful **no-op** — it emits no event, runs no cascade, and never reaches the legality check.
2. Verify the source-state → target-outcome transition is in the allowed table above. If not, fail with a `transitioning` / `invalid_transition` Activity Error.
3. For removal outcomes requiring meta: verify the required field is present. If not, fail with a `transitioning` / `missing_meta` Activity Error (the Primitive `cause` names the missing meta field).
4. Apply the transition through the Status Manager (or whatever the implementation's mutation funnel is — per the API-only access rule in [02 — Conformance](./02-conformance.md#api-only-access)).
5. Emit the corresponding `ChangeEvent`.
6. Run the cascade rules below before returning.

### Adapt — special case

Adapt is the resolution path for a `hanging` atom whose content needs revision. It has no direct `update_status` mapping; it composes a content operation with an explicit status operation:

1. **Revise the content in place** — directly via `Atoms.update(...)`, or via `Atoms.propose(...)` with the corrected content, which the Conflict pipeline folds into the existing atom as a revision ([06 § Revision and succession](./06-conflict-classification.md#revision-and-succession)). The atom's `id` is unchanged; an `Updated` event is emitted; the new `Source` is appended ([03 § Mutability and revision](./03-data-model.md#mutability-and-revision)).
2. **Transition `hanging → active`** via `update_status` — a content revision never changes status, so the resolution is recorded as its own explicit transition.

When the resolving content is a **different claim** — it fails the provenance-fidelity rule ([03](./03-data-model.md#the-provenance-fidelity-rule)) — it enters as a new atom via `propose`, and the hanging atom transitions per [Revision, succession, and the prior atom's status](#revision-succession-and-the-prior-atoms-status).

## Cascade rules

When an atom transitions out of `active` (to any other state or to a removal outcome), the implementation MUST propagate status changes through the snapshot per the rules below.

**The cascade is status-driven.** It fires on status transitions and on nothing else: an in-place content revision (`Updated` — [03 § Mutability and revision](./03-data-model.md#mutability-and-revision)) is not a transition and never hangs a dependent; revision impact is surfaced as advisory dependents in the revision result ([`interfaces/atoms.md`](../interfaces/atoms.md#update)) and analyzed on demand by [Blast Radius](./08-blast-radius.md). There is no hidden middle: content operations are always silent (advisory only); status operations always run the cascade.

The cascade has three independent channels — all three apply on the same transition, in the sequence fixed by [Cascade ordering](#cascade-ordering). Each channel is governed by a different mechanism.

The cascade never resolves what it hangs. A cascaded dependent transitions to `hanging`; its resolution is **per-atom and contextual** — each hanging atom is resolved individually (via `update_status` or a subsequent ingestion), on its own merits and in its own context. A shared trigger does not imply a shared resolution across the affected set.

### Channel 1: predicate-kind cascade

For each **asserted** RelationAtom incident on the changing atom, look up the PredicateAtom (via the RelationAtom's `is_a`) and walk up to its PredicateKindAtom. Apply the cascade rule for that kind:

| Predicate kind | Trigger (on the changing atom) | What hangs |
|---|---|---|
| `dependency` | Changing atom is the **target** (prerequisite); it leaves `active` — any transition to `hanging`, `deferred`, `deprecated`, or a removal outcome | The **source** (dependent) transitions from `active` to `hanging` |
| `enablement` | Changing atom is the **source** (prerequisite); it leaves `active` — same trigger set | The **target** (dependent) transitions from `active` to `hanging` |
| `composition` | Changing atom is the **source** (whole), new state is any removal outcome — AND the RelationAtom's `confidence` ≥ the configured composition cascade threshold (default 0.8) | The **target** (part) transitions from `active` to `hanging` |
| `component_of` | Changing atom is the **target** (whole), new state is any removal outcome — AND `confidence` ≥ threshold | The **source** (part) transitions from `active` to `hanging` |
| `aggregation`, `member_of` | — | No cascade |
| `specialization`, `generalization` | — | No cascade |
| `association` | — | No cascade |
| `equivalence` | — | No cascade — equivalence never transmits a cascade (see [Tree boundaries and cascade](#tree-boundaries-and-cascade)) |

`composition` and its inverse `component_of` mirror each other (whole removed → part hanging, gated by confidence); likewise `dependency` and its inverse `enablement` (prerequisite leaves `active` → dependent hanging). For the strong whole-part kinds, a transition to `hanging` (not removal) does NOT cascade — only removal outcomes do, reflecting their structural-rather-than-epistemic nature.

For `dependency` / `enablement`, the trigger set is **uniform with the scope-premise channel**: any transition of the prerequisite out of `active` — `hanging`, `deferred`, `deprecated`, or a removal outcome — cascades to the dependent. A prerequisite in any of those states is no longer a stable foundation.

For a transition to `superseded`, channel-1 (and channel-2) evaluation applies the resolution invariant — incident references are evaluated against the active successor, so a gated succession transitions nothing ([Revision, succession, and the prior atom's status](#revision-succession-and-the-prior-atoms-status)).

**Channel 1 walks asserted edges only.** Derived RelationAtoms — T2 transitive closures, chain-derived edges ([03 — Data Model § Chain derivation](./03-data-model.md#chain-derivation)), and T1 inverse materializations — never transmit a cascade. Multi-hop propagation emerges from the fixpoint walking asserted edges one hop at a time; a derived shortcut edge would only duplicate work the fixpoint already does, never extend it (and an asserted edge's materialized inverse restates the same claim — the asserted edge's own kind rule covers both directions). Derived atoms participate in the cascade solely as channel-3 invalidation targets.

### Channel 2: Scope premise cascade

Every atom carries `Scope.constraint_atoms: AtomId[]` — a list of ConstraintAtoms whose status gates this atom. When any listed ConstraintAtom transitions to a non-`active` state (any state other than `active`), every atom whose `Scope.constraint_atoms` contains the changing atom's id MUST transition from `active` to `hanging`.

This channel is equivalent to a `kind=dependency` RelationAtom from the gated atom to each listed ConstraintAtom but stored compactly on `Scope` for the common case where many atoms share the same premise constraints. The equivalence is exact: both fire on any transition out of `active`, with the same `hanging` result.

### Channel 3: derivation invalidation

For every derived atom whose `derivation.from` contains the changing atom's id:

- If the source's new state is `superseded` (succession): the derived atom MUST be recomputed with references resolved to the successor ([Observable resolution invariant](#observable-resolution-invariant)); if recomputation no longer produces the same atom, it transitions to `removed`.
- If the source's new state is any other removal outcome: the derived atom MUST be invalidated — transitioned to `removed` (a tombstone, same two-step rule as asserted atoms; GC may purge it later).
- If the source's new state is `hanging` / `deferred` / `deprecated`: the derived atom SHOULD be recomputed; if recomputation no longer produces the same atom, it transitions to `removed`.

Derivation invalidation is automatic — no human review required. The rules above bind the derived atoms a snapshot has **materialized**; where a caching strategy materializes nothing, they are met by recomputation at read time ([04 — Snapshots § Derived atoms within a snapshot](./04-snapshots.md#derived-atoms-within-a-snapshot)). A channel-3 invalidation of a materialized derived atom is recorded as a `DerivedInvalidated` event (carrying the invalidated atom, the triggering source, and the derivation rule) — **not** `StatusChanged`; derived-atom records are advisory and sit outside the stream's replayable contract ([05 — Delta Streams § Derived atoms in the stream](./05-delta-streams.md#derived-atoms-in-the-stream)). See [Cascade provenance emissions](#cascade-provenance-emissions).

### Cascade ordering

When an atom transitions, the implementation MUST apply cascades in this order:

1. Derivation invalidation (channel 3) — fastest; removes stale derived atoms before further reasoning runs.
2. Predicate-kind cascade (channel 1) — applies kind-specific propagation.
3. Scope premise cascade (channel 2) — applies the constraint_atoms gate.

After cascades 1–3 complete, the implementation MUST run a fixpoint: if any atom transitioned in this round, re-run cascades 1–3 with the newly-transitioned atoms as the changing set. Continue until no atom transitions in a round. The fixpoint terminates because, within a single cascade run, an atom that has already transitioned is not transitioned again — cascades only move atoms out of `active`, so the transitioned set grows monotonically and is bounded by the snapshot's atom count.

### The cascade is the single source of truth

The cascade rules in this section (the three channels, the per-kind cascade table, and the composition confidence gate) are the **one** normative definition of impact propagation. No other component may define cascade behavior independently:

- **Atoms** applies it as the *mutating* run when a transition commits (above).
- **Atoms** exposes the *read-only dry-run* of the very same engine as `simulate_transition(atom_id, hypothetical)` ([`interfaces/atoms.md`](../interfaces/atoms.md)) — it computes the fixpoint without mutating.
- **[Blast Radius](./08-blast-radius.md)** obtains its structural impact set from `simulate_transition`; it MUST NOT re-implement the traversal.

**Equivalence invariant (normative):** for any transition `T = (atom_id, hypothetical)`, the structural impact set of `simulate_transition(T)` MUST equal the set of atoms that committing `update_status(T)` would actually transition, with the same predicted state per atom. The dry-run and the real run are one engine; they MUST NOT diverge. (Blast Radius's optional LLM-implicit pass is *additive* and explicitly outside this equivalence.)

### Cascade provenance emissions

Every transition cascaded through channels 1–2 MUST emit a `ChangeEvent` of kind `StatusChanged` whose payload carries **structured cascade provenance** — the fields `cascade_channel`, `cause_atom_id`, and (for channel 1) `predicate_kind`; the payload shape is declared in [`interfaces/atoms.md`](../interfaces/atoms.md):

| Channel | `cascade_channel` | `cause_atom_id` | `predicate_kind` |
|---|---|---|---|
| Predicate-kind (1) | `"predicate-kind"` | The upstream atom whose transition triggered this one | The kind of the mediating RelationAtom's predicate |
| Scope premise (2) | `"scope-premise"` | The gating ConstraintAtom | absent |

Channel-3 invalidations emit `DerivedInvalidated` — payload: the invalidated atom, the triggering `source`, and the derivation `rule` — not `StatusChanged` (advisory, emitted for materialized derived atoms; [05 — Delta Streams § Derived atoms in the stream](./05-delta-streams.md#derived-atoms-in-the-stream)).

The human-readable `rationale` string on a cascaded event is informative. Consumers MUST NOT parse it for control flow or rendering — the structured payload fields are the contract, consistent with the prohibition on string-matching in [10 — Error Model](./10-error-model.md).

## Atoms in removal outcomes

A **previously-committed** atom that transitions to a removal outcome becomes a **tombstone** — it remains in the snapshot until an optional GC pass purges it:

- MUST NOT be a source or target of any active cascade channel — it cannot transmit further cascades after its own removal cascade completes. (References *through* a `superseded` tombstone resolve to its successor per the [Observable resolution invariant](#observable-resolution-invariant); the tombstone itself transmits nothing.)
- MUST NOT be returned by `Atoms.list_scope(scope, ["active", "hanging", "deferred", "deprecated"])` (the default present-state filter).
- MUST be returned by `Atoms.list_scope(scope, ["superseded", "duplicate", "removed"])` until purged by GC.
- Carries the full removal record: `status.state`, `status.reason`, `status.correlation_id` (the operation that removed it), and `status.meta` (`superseded_by` / `duplicate_of`, plus the conflict-pipeline classification — `outcome_kind` / `outcome_stage` / `outcome_confidence` — when a pipeline resolution drove the transition; [06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).
- Dead incoming references (other atoms pointing at a removed atom) are valid — the cascade fired at removal time; subsequent references resolve to a tombstone (and, after GC purge, to a dead reference) per [12 — Edge Cases](./12-edge-cases.md).

A **never-committed** proposal is not a tombstone: a `rejected` proposal (and any hard-blocked one) was never added to the snapshot, is **not** returned by `list_scope` for any status filter, and exists only as a `Rejected` audit event carrying the candidate ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).

## Revision, succession, and the prior atom's status

A content revision keeps the atom and its `id` — the provenance-fidelity rule decides whether a change qualifies ([03 § Mutability and revision](./03-data-model.md#mutability-and-revision)). When a **different claim** enters instead, the prior atom's status is resolved by what happened to the old claim:

| The old claim… | Prior atom | Dependents |
|---|---|---|
| Still operative — the two claims coexist | Stays `active` (a revision MAY narrow its scope); an already-`hanging` atom whose coexistence is accepted moves to `deferred` | None hang; an already-cascaded set resolves per atom |
| Phasing out — holds for existing dependents, closed to new ones | `deprecated` | Cascade fires; dependents re-validate ([Cascade rules](#cascade-rules)) |
| Rescinded — no longer holds | `removed` | Cascade fires |
| **Fully absorbed by a compatible refinement** | `superseded` (`meta.superseded_by` = the successor) | None — the silent succession path (below) |

### The succession gate

`superseded` records **claim succession**: a different claim fully absorbs this one as a **compatible refinement** — every reading the superseded atom supported remains supported by its successor. The gate is binding: an atom MUST NOT be superseded by a *contrary* successor — silently re-pointing referencers at a meaning-changed claim is not a resolution. Contrary replacement goes through `deprecated` / `removed`, whose cascade forces dependents to re-validate. Where the Conflict pipeline classifies the replacement, it applies the gate ([06 § Revision and succession](./06-conflict-classification.md#revision-and-succession)); a direct `update_status(..., "superseded", ...)` places the same obligation on the reviewer.

### Observable resolution invariant

When an atom is `superseded`, asserted references to its id MUST resolve to its **current successor** — following `meta.superseded_by` chains to the newest non-superseded atom — in every read surface, relation traversal, entailment computation, and cascade evaluation. Direct addressing is exempt: `get_by_id` on the superseded id returns the tombstone itself; the invariant governs references *from other atoms*. The mechanism — virtual chain-following, redirect records, or physical re-pointing of stored references — is implementation-specific; the resolution behavior is the contract.

This invariant is what makes succession silent. A `superseded` transition runs the cascade like any other transition out of `active`, but channels 1 and 2 evaluate incident asserted edges and scope-premise references against the successor; because the gate guarantees the successor is a compatible refinement and it is `active`, no dependent loses its premise and nothing hangs. (Equivalently: `simulate_transition(a, "superseded")` yields an empty structural impact set — see [`interfaces/atoms.md`](../interfaces/atoms.md#simulate_transition).) Channel 3 still applies: derived atoms premised on the superseded atom are recomputed with references resolved to the successor ([Channel 3](#channel-3-derivation-invalidation)).

## Garbage collection

Garbage collection is an **optional** maintenance operation that physically purges tombstoned atoms (those in a removal-outcome state). It is the **only** path to physical deletion — no other operation deletes an atom; a never-committed proposal leaves no tombstone, so there is nothing for GC to touch.

- GC is **distinct from the removal transition**: it runs later, on its own schedule, never as part of the `update_status` that created the tombstone.
- **Tombstones are visible from transition through commit.** GC MUST NOT purge a tombstone from the working snapshot in which its removal transition occurred before that snapshot is published or discarded — the tombstone is part of the reviewable diff ([04 — Snapshots § Removal outcomes and snapshots](./04-snapshots.md#removal-outcomes-and-snapshots)). Post-publication retention, scheduling, and compaction are implementation-specific.
- GC is a `system:`-correlated operation (see [05 — Delta Streams](./05-delta-streams.md)); each purge emits a `Purged` change event, so the physical deletion is itself recorded.
- GC MUST NOT purge a tombstone the snapshot still needs to honor its contracts — in particular a `superseded` tombstone through which asserted references still resolve ([Observable resolution invariant](#observable-resolution-invariant)) — unless the implementation preserves the behavior by other means (e.g., re-pointing stored references before the purge). Beyond that, the retention policy (age, reference-reachability, depth) is a deployment choice.
- The append-only Delta Stream retains the removal transition regardless of GC, so an atom's removal record survives even after its tombstone is purged.

## Composition cascade — confidence gate

The composition cascade fires only when the composition RelationAtom's `confidence` is at or above the configured threshold (default 0.8, deployment-overridable). Below the threshold, the source's removal is recorded but the target stays `active` — extractor confidence wasn't strong enough to justify cascading destruction of data downstream.

When confidence is below threshold but composition is otherwise valid, the cascade run MUST emit a `CompositionCascadeSkipped` event — payload in [`interfaces/atoms.md § changes_since`](../interfaces/atoms.md#changes_since): the mediating relation, the removed whole (`cause_atom_id`), the spared part (`spared_atom_id`), the relation's `confidence`, and the `threshold` in force. The event is advisory: it records that the gate suppressed the cascade and applies as a no-op to materialized state. A reviewer acts on it through the ordinary surfaces — transition the spared part explicitly via [`update_status`](../interfaces/atoms.md#update_status), or revise the relation's `confidence` via [`Atoms.update`](../interfaces/atoms.md#update) so future cascade evaluations clear the gate.

## Lifecycle and derivation

Derived atoms (those with a populated `derivation` field) follow the same state machine but with these constraints:

- Derived atoms are typically created as `active` and stay `active` until invalidated.
- A derived atom MUST NOT be explicitly transitioned via `Atoms.update_status` — only the entailment engine (or its invalidation) changes its status.
- A derived atom MAY collide with an asserted atom on `(subject, predicate-via-is_a, object, scope)`. In that case the asserted atom wins; the derived atom is removed (per [03 — Data Model](./03-data-model.md)).

## Tree boundaries and cascade

A tree is an autonomous existence — its own stakeholders, owners, and lifecycle ([03 — Data Model § Scope](./03-data-model.md#scope)). The cascade interacts with tree boundaries as follows:

- **Asserted edges cascade wherever they point.** The predicate-kind cascade follows RelationAtom edges regardless of tree: an asserted cross-tree edge (e.g., a `dependency` from a design-tree decision to an implementation-tree constraint) is a deliberate claim coupling the two systems and cascades like any within-tree edge.
- **Scope-premise references likewise.** `Scope.constraint_atoms` entries are followed regardless of which tree holds the gating constraint — the channel's dependency equivalence carries over.
- **Derivation invalidation follows `derivation.from` regardless of tree.** A derived atom whose premises span trees is invalidated when any premise leaves `active`.
- **`kind=equivalence` never transmits a cascade.** An equivalence link maps similar concepts across autonomous systems; it records correspondence, not coupling. No automatic transition ever crosses a tree boundary via `kind=equivalence` — each tree's owners decide whether and how a twin concept follows. [Blast Radius](./08-blast-radius.md#cross-tree-behavior) surfaces equivalence-linked twins of impacted atoms as **advisory** entries, alongside — never inside — the structural impact set.

Tree partitioning controls Duplicate detection and disjointness ([06](./06-conflict-classification.md)). Status propagation across trees happens only through asserted structure — the edge or premise reference itself is the deliberate coupling.
