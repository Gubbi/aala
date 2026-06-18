# 12 — Edge Cases

This chapter binds behavior at the boundaries of the model: dead references, references to tombstones, subsumption cycles, malformed input, concurrency races, schema evolution on live data, derived-atom visibility, partial failures, and convergence. Each section is the normative home of the edge-case rule it states and cites the chapter that owns the underlying machinery ([01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership)); shapes and per-method declarations stay in the interface contracts.

## Dead AtomId references

A proposed atom references an AtomId that resolves to **nothing** in the current snapshot — the referent was never committed, or its tombstone has been purged by garbage collection ([07 § Garbage collection](./07-atom-lifecycle.md#garbage-collection)). A purged id is never re-minted ([Appendix A § Identifier minting](./appendix-a-registries.md#identifier-minting)), so an absent referent either arrives later (out-of-order ingestion) or never will; the rules below do not distinguish — review does.

The anchor reference is structural; every other reference is admitted and flagged ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes)):

| Reference carrying the dead AtomId | Behavior |
|---|---|
| `Atom.is_a` | Reject the proposal with an `extracting` / `schema_violation` Activity Error (the Primitive `cause` carries `field: "is_a"`, `reason: "target_not_found"`). The anchor is the structural side of the schema-violation fence: without it the atom cannot be interpreted at all, so `is_a` **MUST** resolve to a present-state atom at proposal time. |
| `RelationAtom.subject` or `RelationAtom.object` | Admit the relation; it enters `active`, flagged `DeadEdge` (soft-prompt — [Appendix A § Conflict outcomes](./appendix-a-registries.md#conflict-outcomes)). Until the referent materializes the dead end is inert: the edge transmits no cascade through it, and referent-dependent validation (the predicate's `domain_classifications` / `range_classifications`) is held with the flag. |
| `subject` reference of a referential atom | Admit the atom, flagged `DeadEdge`. Referent-dependent validation — the classification's `domain_classifications` check (`SubjectClassificationViolation`) — is held with the flag. The atom is queryable and traversable meanwhile. |
| `Scope.constraint_atoms[]` entry | Admit the atom, flagged `DeadConstraint` (soft-prompt). The scope-premise channel ignores the dead entry — the unknown premise gates nothing — until the referent materializes ([07 § Channel 2](./07-atom-lifecycle.md#channel-2-scope-premise-cascade)). |
| `Atom.derivation.from[]` entry | **MUST NOT** happen for honest derivation — `derivation` is written only by the entailment engine. If detected, treat as data corruption: invalidate the derived atom — transition it to `removed` — and emit a `DerivedInvalidated` event whose `source` carries the unresolvable id ([07 § Channel 3](./07-atom-lifecycle.md#channel-3-derivation-invalidation)). |

**Materialization.** When a previously-dead referent is later added to the snapshot, the implementation **MUST**, within the operation that adds it:

1. Clear the `DeadEdge` / `DeadConstraint` record — materialization is the record's defined resolution ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).
2. Run the validation that was held with the flag (the referent-dependent checks above), surfacing any conflict outcome per the registry.
3. Evaluate the now-live structure's cascade triggers against the referent's **current** status, exactly as [Premise state at admission](#premise-state-at-admission) prescribes — a referent that enters `active` gates nothing further.

## References to removal-outcome tombstones

A reference can also name an id that resolves to a **tombstone** — a previously-committed atom in a removal-outcome state ([07 § Atoms in removal outcomes](./07-atom-lifecycle.md#atoms-in-removal-outcomes)). (`rejected` proposals were never committed and leave no tombstone; their ids fall under [Dead AtomId references](#dead-atomid-references).)

| Referent | Behavior |
|---|---|
| `superseded` tombstone — any reference | The reference resolves to the **current successor** ([07 § Observable resolution invariant](./07-atom-lifecycle.md#observable-resolution-invariant)) and is treated as a reference to that present-state atom: validation, entailment, and trigger evaluation all run against the successor. |
| `duplicate` / `removed` tombstone, named by `Atom.is_a` | Reject with an `extracting` / `schema_violation` Activity Error (`field: "is_a"`, `reason: "target_not_in_present_state"` on the Primitive `cause`). The anchor **MUST** resolve to a present-state atom, and a non-`superseded` tombstone has no successor to resolve through. |
| `duplicate` / `removed` tombstone, named by a relation endpoint, a referential `subject`, or a `Scope.constraint_atoms[]` entry | Admit. References to tombstones are valid ([07 § Atoms in removal outcomes](./07-atom-lifecycle.md#atoms-in-removal-outcomes)) — a claim about a withdrawn claim is representable, and the `conflicts_with` linkage machinery itself relies on it. The tombstone transmits and receives no cascade; where the admitted structure is cascade-bearing, its trigger is evaluated at admission against the tombstoned state ([Premise state at admission](#premise-state-at-admission)) — a withdrawn premise hangs its dependent immediately. A `duplicate` tombstone's `meta.duplicate_of` names the natural re-pointing target for review. |

Pre-existing RelationAtoms pointing at an atom that transitions to a removal outcome are governed by the cascade ([07 — Atom Lifecycle](./07-atom-lifecycle.md)): they are never retroactively rejected; the cascade transitions what the kind rules say it transitions, and the surviving references thereafter resolve to the tombstone.

## Premise state at admission

The cascade fires on status transitions ([07 § Cascade rules](./07-atom-lifecycle.md#cascade-rules)) — but the structure that carries a cascade can be admitted **after** the triggering transition already happened. To keep the committed state independent of arrival order, admission evaluates triggers against present state:

When a proposal admits cascade-bearing structure — a RelationAtom of a cascading kind, or an atom whose `Scope.constraint_atoms` gate it — the implementation **MUST** evaluate the structure's cascade trigger against the current status of the atoms it references, within the proposing operation. If the trigger condition already holds — the prerequisite or premise has left `active`: it is `hanging`, `deferred`, `deprecated`, or a non-`superseded` tombstone — the kind's consequence applies: the designated dependent transitions to `hanging` with ordinary cascade provenance ([07 § Cascade provenance emissions](./07-atom-lifecycle.md#cascade-provenance-emissions)), and the fixpoint runs. The resulting state **MUST** equal the state that would have been reached had the structure been present when the triggering transition committed.

- The rule reaches **pre-existing dependents**: admitting a `dependency` edge from an `active` atom to a `hanging` prerequisite hangs the (pre-existing) dependent. The new edge asserts the coupling; the evaluation honors it.
- The composition confidence gate applies as at any cascade evaluation ([07 § Composition cascade — confidence gate](./07-atom-lifecycle.md#composition-cascade--confidence-gate)).
- An **absent** referent is held in abeyance instead (`DeadEdge` / `DeadConstraint`, [Dead AtomId references](#dead-atomid-references)); the evaluation runs at materialization.
- A `deprecated` premise hangs **new** dependents under this rule while existing dependents stand — exactly `deprecated`'s meaning: still holds for existing dependents, closed to new ones ([07 § States](./07-atom-lifecycle.md#states)).
- The direct revision surface is **not** an admission: revising a premise into an existing atom (an `update` that changes `scope` or a relation endpoint) transitions nothing — content revisions are silent, with dependents reported as advisory ([03 § Revisions never cascade](./03-data-model.md#revisions-never-cascade)); [Blast Radius](./08-blast-radius.md) assesses the impact on demand.

## Cycles in `is_a`

The subsumption hierarchy **MUST** remain acyclic: a cycle makes the cumulative-schema walk and the T1/T2 closures non-terminating, and [03 — Data Model](./03-data-model.md#per-type-field-requirements) requires every classification's `is_a` chain to terminate at the root. Accordingly:

- **Detection is complete and unconditional.** A proposal that closes a subsumption cycle — at **any** depth — **MUST** be detected and classified. There is no detection depth bound. (Informative: walking the proposal's `is_a` chain to the root is linear in chain length; explicit-edge cycle detection is an incremental reachability check.)
- **Classification follows the registry** ([Appendix A § Conflict outcomes](./appendix-a-registries.md#conflict-outcomes)): a cycle that runs through at least one `is_a` field link — including an `is_a` 2-cycle and mixed `is_a`/edge 2-cycles — is `GeneralizationCycle` (hard-block) at any depth; a 2-cycle of explicit `kind=specialization` edges alone is `GeneralizationEquivalenceConflict` (soft-prompt, with its enumerated resolution paths); explicit-edge cycles of depth ≥ 3 are `GeneralizationCycle`.
- **Other asymmetric kinds get the same completeness.** Directed cycles of asserted edges in the asymmetric kinds surface as `AsymmetryViolation` (hard-block); the hard characteristics are unconditional, so their detection **MUST NOT** be depth-bounded either.
- **Detection runs against the current snapshot only.** A proposal is not preemptively rejected because some future change *could* complete a cycle through it; the change that would close the cycle is rejected at its own proposal or revision time.
- **The revision surface is covered.** An `Atoms.update` whose `changes.is_a` would close a cycle leaves the atom's own `is_a` chain unable to resolve to the root — its schema no longer resolves — so the revision gate rejects it before mutation with `revising` / `schema_violation` ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes), the revision clause).

## Malformed input

Each "Reject" below is an `extracting` / `schema_violation` Activity Error — the structural side of the schema-violation fence ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes); registry row in [10 — Error Model](./10-error-model.md#activity-end-states)); the noted `field` / `reason` travel as diagnostics on the Primitive `cause`.

| Situation | Behavior |
|---|---|
| Atom proposed with unrecognized `type` value | Reject (`field: "type"`, `reason: "unknown_atom_type"`). |
| Atom whose `id` is malformed, or whose `<type>` id prefix disagrees with its `type` field | Reject (`field: "id"`, `reason: "malformed_id"` / `"id_type_mismatch"`). |
| Atom missing a required field for its type (e.g., `entity` without `is_a`) | Reject (`field: "<name>"`, `reason: "required"`). |
| Atom with both `derivation` populated AND `sources` non-empty | Accept. A derived atom carries sources inherited from its premises ([03 § Provenance](./03-data-model.md#provenance--source)); the combination is well-formed. |
| Atom whose `confidence` is outside `[0.0, 1.0]` | Reject (`field: "confidence"`, `reason: "out_of_range"`). |
| Atom whose `Scope.tree` does not match a registered tree | Reject (`field: "scope.tree"`, `reason: "unregistered_tree"`). |
| ClassificationAtom (non-root) with no `is_a` | Reject (`field: "is_a"`, `reason: "required_except_root"`). Only the single entity-domain root may omit `is_a`. |

Violations of a *referenced definition's* rules sit on the logical side of the fence: they are **conflict outcomes**, not `schema_violation` Activity Errors — a classification's `subject_kind` contradicting an ancestor (`SubjectKindInheritanceConflict`), a predicate's characteristics contradicting its kind (`CharacteristicKindConflict`), a `subject == object` self-loop on an irreflexive kind (`IrreflexivityViolation`). See the [Outcome registry](./06-conflict-classification.md#outcome-registry).

## Concurrency

| Race | Resolution |
|---|---|
| Two `Atoms.propose(...)` calls for the same `fragment.message_id` | Writes serialize at operation grain ([04 § Operation isolation](./04-snapshots.md#operation-isolation-and-concurrency)); the second call is a replay under the method's idempotency key — a full no-op returning the recorded `ProposeResult` ([00-shared-types § Idempotency](../interfaces/00-shared-types.md#idempotency)). No duplicate state is produced. |
| Two `update_status(...)` calls on the same atom | Serialized at operation grain ([00-shared-types § Concurrency](../interfaces/00-shared-types.md#concurrency)): implementations MAY queue (blocking) or fail one with a `contended` Activity Error (`infra`/`retryable`; caller retries). A queued call applies against the post-first state: an equivalent repeat is the idempotent no-op; a different outcome is an ordinary transition validated against the post-first state by the transition table ([07 § API for transitions](./07-atom-lifecycle.md#api-for-transitions)) — after a first call lands a terminal removal outcome, a second call requesting `hanging` fails `transitioning` / `invalid_transition`; it does not "win". |
| Explicit `update_status` while a cascade is running | The cascade belongs to the operation that triggered it — one atomic unit ([10 § Atomicity boundaries](./10-error-model.md#atomicity-boundaries)). The explicit call is its own operation: it serializes behind the cascading one and applies against the post-cascade state. Restoring a cascaded-`hanging` atom to `active` is then the ordinary per-atom resolution, legal per the transition table. |
| Classification schema revision concurrent with a proposal that validates against it | Both are write operations on one snapshot and serialize; validation runs against the schema state at the proposal operation's admission — the revised schema, when the proposal queued behind the revision. An implementation that rejects rather than queues fails one with `contended`; the caller retries against the settled schema. |
| Atom transition concurrent with Blast Radius analysis | `analyze` is read-only and consumes a consistent snapshot view; the stored report is a point-in-time artifact. A transition committed after generation makes the report stale for re-analysis — a repeat `analyze` then opens a fresh report ([08 § Determinism and report identity](./08-blast-radius.md#determinism-and-report-identity)). |
| Two sessions publish working snapshots forked from the same canonical (publish race) | The first compare-and-swap on the canonical pointer wins. The second publish fails with a `snapshotting` / `non_fast_forward` Activity Error (`client`/`user_action`); that publisher re-forks from the new canonical and replays its work (idempotency on `message_id` makes replay safe). Snapshot merge is out of scope for `aala-v0.1` ([04 § Canonicalization](./04-snapshots.md#canonicalization)). |
| `switch_snapshot` concurrent with the same session's in-flight call | The in-flight call completes against the binding in effect at its admission; the rebind applies to calls admitted after it returns ([04 § Session binding](./04-snapshots.md#session-binding)). |

## Schema evolution on a live tree

A `ClassificationAtom.schema` changes through a **governed revision** — an `Atoms.update` carrying a `schema` change, validated against the inheritance rules and emitting `ClassificationSchemaUpdated` after the `Updated` event ([03 § Schema-carrying atoms](./03-data-model.md#schema-carrying-atoms)). A revision is a content operation: it transitions no statuses ([03 § Revisions never cascade](./03-data-model.md#revisions-never-cascade)). Consequences for atoms already classified under the revised chain surface through the pipeline's recompute trigger ([06 § When the pipeline runs](./06-conflict-classification.md#when-the-pipeline-runs)) as **recorded conflict situations** — never as automatic transitions:

| Schema change | Behavior |
|---|---|
| Tightens the cumulative schema — adds a `required_relations` entry, adds a `required: true` attribute, narrows an attribute's `value_type` | The revision applies; nothing transitions. The recompute re-validates atoms classified under the revised chain against the new cumulative schema and records the matching outcome — `MissingRequiredRelation`, `MissingRequiredAttribute`, `AttributeTypeViolation` — in atom-space for review ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)). New proposals validate against the revised schema. |
| Removes or loosens entries | Every atom that satisfied the old cumulative schema satisfies the new one; nothing surfaces. |
| Fixes an undeclared `subject_kind` (abstract → `named`, or abstract → `referential`) | Legal where no descendant already fixed it differently — the governed-mutation validation enforces the inheritance rules and rejects a contradicting revision with `revising` / `schema_violation`. Classified atoms whose `subject` does not match the now-fixed kind are recorded as `InvalidSubject` situations for review. |
| Flips a **declared** `subject_kind` (`named` ↔ `referential`) | Never a revision. The two are disjoint subject treatments, not a narrowing: the flip changes the meaning of every existing subject — a different claim about the class, failing the provenance-fidelity rule ([03 § The provenance-fidelity rule](./03-data-model.md#the-provenance-fidelity-rule)). The revision gate **MUST** reject it (`revising` / `schema_violation`). The route is succession by review: a new ClassificationAtom carrying the new declaration enters, instances are reclassified, and the old classification is phased out — `deprecated`, then a removal outcome — so its dependents re-validate ([07 § Revision, succession, and the prior atom's status](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status)). |

A default resolution mode binds **proposals** ([06 § Resolution modes](./06-conflict-classification.md#resolution-modes)): hard-block rejects a candidate. A situation detected among already-committed atoms — the recompute path above — has no candidate to reject; the pipeline records it in atom-space regardless of the outcome's default mode, and review resolves it through the ordinary surfaces ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).

`PredicateAtom.schema` revisions follow the same pattern (`PredicateSchemaUpdated`), with the chain/kind restrictions additionally validated at the gate ([03 § Chain derivation](./03-data-model.md#chain-derivation)).

## Derived atoms at the read and result surfaces

Derived state is a deterministic function of asserted state, and what is materialized at any moment is a per-snapshot caching choice ([04 § Derived atoms within a snapshot](./04-snapshots.md#derived-atoms-within-a-snapshot)). The Delta-Stream consequence is settled — derived-atom events are advisory ([05 § Derived atoms in the stream](./05-delta-streams.md#derived-atoms-in-the-stream)); this section settles the same freedom where it meets reads and operation results:

- **Derivable means readable, identically.** Within one snapshot, every read that admits derived atoms **MUST** present the same derived set for the same asserted state, whatever the caching strategy — a lazy implementation computes at read time. A derived atom's identity is stable within the snapshot: recomputing the same derivation yields the same `AtomId`, so `get_by_id`, `derivation.from` entries, and `derived_loss` lists stay meaningful across recomputations.
- **Invalidated derivations diverge lawfully.** After a channel-3 invalidation ([07 § Channel 3](./07-atom-lifecycle.md#channel-3-derivation-invalidation)), an eagerly materializing implementation holds a `removed` tombstone for the derived atom until GC purges it; a lazy implementation holds nothing. `get_by_id` on the invalidated derivation's id returns the tombstone in the former and `null` in the latter — **both are conformant**. Consumers **MUST** treat the two answers alike (the derivation is not part of present state) and **MUST NOT** depend on observing derived-atom tombstones, their `Purged` events, or any other materialization artifact.
- **Result fields mirror materialization.** `ProposeResult.derived_added` and `UpdateResult.derived_added` / `derived_invalidated` ([`interfaces/atoms.md`](../interfaces/atoms.md#propose)) report the operation's **materialization activity**, exactly as the advisory stream events do: under lazy caching they MAY be empty even when the operation changed what is derivable. A consumer that needs derived consequences recomputes them from asserted state; the fields are hints, never the contract.

## Tree lifecycle

A `TreeId` is **immutable** — the same principle as the stable `AtomId` ([03 — Data Model](./03-data-model.md)). Atoms reference the tree via `Scope.tree`; those references never rewrite, and no bulk-migration operation exists.

| Operation | Behavior |
|---|---|
| Display rename | A tree's `display_name` and `description` are mutable via [`Orchestration.update_tree`](../interfaces/orchestration.md#update_tree). The change emits a `TreeRenamed` event carrying the tree id and the changed display fields — never an id swap. Atoms are untouched; `Scope.tree` values do not change. |
| Tree merge / split | **Out of scope for `aala-v0.1`** — no merge or split operation is defined. Restructuring knowledge across trees happens through ordinary ingestion and review: new atoms proposed in the target tree, removal outcomes applied in the source tree, with cross-tree `kind=equivalence` links recording correspondence where both survive. |

## Snapshot transitions

| Transition | Behavior |
|---|---|
| `working` snapshot promoted to `canonical` | The promotion is atomic: the compare-and-swap on the canonical pointer, the state change to `canonical`, and the `SnapshotPublished` emission are one step — observers see either the pre or the post state. The previously canonical snapshot MUST be an ancestor of the promoted snapshot (the fast-forward invariant; otherwise the publish fails `non_fast_forward`), and it keeps state `canonical` as immutable history ([04 § Canonicalization](./04-snapshots.md#canonicalization)). The promoted snapshot's `parent` is unchanged by promotion — lineage records the fork, not the pointer movement. |
| `working` snapshot discarded | All atoms with provenance only in the discarded snapshot are unreachable via canonical reads. Implementations MAY garbage-collect them or retain them for forensic purposes per the retention policy. Sessions still bound to the discarded snapshot fail subsequent calls with `snapshotting` / `discarded` until they rebind ([04 § Session binding](./04-snapshots.md#session-binding)). |
| `canonical` snapshot rolled back to an earlier `canonical` | The canonical pointer never rewinds. A rollback is a new snapshot built from existing primitives: fork from the **current** canonical, transition the atoms absent from the earlier snapshot to a removal outcome (`removed`) with rationale `"rollback to <snapshot_id>"`, and publish — a fast-forward whose state mirrors the earlier snapshot. |
| Forking from any snapshot | Creates a new `working` snapshot whose `parent` is the source. Atoms in the source are visible in the fork. Changes in the fork don't affect the source until promotion. |

## Convergence — replay equivalence

The replay-equivalence invariant and the causal-ordering rules are owned by [05 — Delta Streams](./05-delta-streams.md#replay-equivalence); this chapter adds only their boundary readings:

- **Dead references in a replayed state are faithful, not failures.** Causal ordering guarantees an in-order consumer never observes a reference the producer had resolved ([05 § Causal ordering](./05-delta-streams.md#causal-ordering-within-a-stream)) — but the asserted state itself may contain flagged dead references ([Dead AtomId references](#dead-atomid-references)). A replayed state containing them matches the source exactly.
- **Derived divergence is lawful.** Equivalence covers asserted state; two consumers — or a consumer and the producer — MAY hold different materialized derived sets over identical asserted state ([05 § Derived atoms in the stream](./05-delta-streams.md#derived-atoms-in-the-stream); [Derived atoms at the read and result surfaces](#derived-atoms-at-the-read-and-result-surfaces)).
- **Every event boundary is a valid checkpoint — including ones the producer never served.** Operations commit atomically ([10 § Atomicity boundaries](./10-error-model.md#atomicity-boundaries)), but the stream delivers their effects event by event, so a consumer that pauses mid-run holds an intra-operation state no producer read would ever return. That state is a correct replay position, not an error; a consumer that wants operation-atomic application MAY buffer one correlation run — contiguous within a snapshot ([05 § Correlation across streams](./05-delta-streams.md#correlation-across-streams)) — and apply it as a unit.

## Partial failures

| Failure | Behavior |
|---|---|
| Crash inside an atomic write operation (`propose`, `update`, `update_status`) | The operation's atomicity boundary covers the whole call, cascade fixpoint and entailment recompute included ([10 § Atomicity boundaries](./10-error-model.md#atomicity-boundaries)): a crash before commit leaves the snapshot unchanged — no events, and nothing recorded to replay ([10 § Idempotency contracts](./10-error-model.md#idempotency-contracts)). The caller observes an error or timeout and retries; for key-identified methods the retry is safe under the idempotency registry ([00-shared-types § Idempotency](../interfaces/00-shared-types.md#idempotency)). An implementation MAY journal and resume internally, provided no intermediate state is ever observable. |
| Conflict pipeline crash mid-evaluation | The `propose` fails (a `classifying_conflicts` pair from the registry, or a universal end-state such as `unavailable`); per the atomicity boundary nothing is committed, and the retry executes as a fresh attempt. |
| Derivation engine crash mid-recomputation | Within a write operation: covered by the first row. Outside one (a lazy recomputation at read time): derived state is recomputable by definition — the failed read fails with its declared pairs, and a retry recomputes ([05 § Derived atoms in the stream](./05-delta-streams.md#derived-atoms-in-the-stream)). Derived atoms are never the record, so no durable damage is possible. |
| State/stream divergence after a crash | A committed operation's state effects and its Delta-Stream events are one commit ([04 § Session binding](./04-snapshots.md#session-binding)): a committed operation whose events are missing — or delivered events whose operation never committed — **MUST NOT** be observable. An implementation persisting state and stream separately MUST reconcile the two during recovery, before serving reads or streams for the snapshot. |
| Partial failure of a composed `Orchestration.ingest` | Atomicity is per-container; recovery and retry semantics are owned by [10 § Cross-container failures](./10-error-model.md#cross-container-failures) — the accepted fragment persists, and a retry replays completed steps as no-ops and resumes incomplete ones. |

## Versioning at runtime

| Situation | Behavior |
|---|---|
| Caller selects an interface version older than the implementation also declares | The implementation MUST honor the older version's contract ([11 § Version negotiation](./11-versioning.md#version-negotiation)). Backward-compatible additions from newer versions are simply absent from the caller's view. |
| Caller selects an interface version the implementation does not declare | The call MUST fail with the universal `unsupported_version` end-state on the called method's activity ([10 — Error Model](./10-error-model.md#activity-end-states)), carrying a version-mismatch reason. The caller downgrades or fails. |
| Atom carrying fields the reader doesn't recognize | The reader MUST preserve unknown fields when re-emitting the atom (e.g., when transiting through a proxy) — forward compatibility per [11 — Versioning](./11-versioning.md#forward-compatibility). |

## Glossary projection edge cases

| Situation | Behavior |
|---|---|
| Two ClassificationAtoms in the same tree with overlapping `aliases` | Stage 2 flags `AliasCollision` (soft-prompt — [Appendix A § Conflict outcomes](./appendix-a-registries.md#conflict-outcomes)): the earlier registration keeps the alias; the later proposal enters flagged for review. The glossary resolves the contested alias to the earlier registration until the record is resolved. |
| EntityAtom whose name (subject) collides with a ClassificationAtom name | No conflict — entities and classifications occupy different namespaces in the glossary projection. The glossary renders both. |
| ClassificationAtom with an empty `definition` and no aliases | Renders in the glossary as a bare class name. Implementations SHOULD surface a review prompt to add a definition; no conflict outcome exists for a missing definition. |
