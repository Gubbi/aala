# 06 — Conflict Classification

The Conflict pipeline examines proposed atoms (and entailment-derived states) and **classifies** what it finds as **conflict outcomes**. An outcome is a classification, not an entity: it names the situation (a kind from the [Outcome registry](#outcome-registry)), lists the involved atoms, and declares a default **resolution mode** — it has no identity or lifecycle of its own. The pipeline is the single channel through which the system identifies and routes ambiguity, duplication, and contradiction; everything it records lives in atom-space — statuses and metadata on the affected atoms, plus linkage RelationAtoms for pending situations ([Outcome recording](#outcome-recording)).

## When the pipeline runs

| Trigger | Examples |
|---|---|
| Atom proposal | `Atoms.propose(fragment, ...)` produces a new atom that may collide with existing ones |
| Lifecycle transition | `Atoms.update_status(...)` moves an atom out of `active`; may surface dependent atoms that need resolution |
| Content revision | `Atoms.update(...)` revises logically significant fields; the entailment recompute it triggers re-runs the T3 checks against the revised structure ([03 § Mutability and revision](./03-data-model.md#mutability-and-revision)) |
| Entailment recomputation | T3 disjointness / cardinality checks (see [03 — Data Model](./03-data-model.md)) fire when derived inferences contradict |
| Cross-tree similarity scan | Implementations MAY periodically scan for high-similarity atoms across trees to suggest equivalence edges |

Every effect the pipeline produces is recorded through ordinary atom events carrying the outcome classification ([Outcome recording](#outcome-recording)).

## Malformed input vs. conflict outcomes

This section is the normative home of the **schema-violation fence** — the boundary between schema-violation Activity Errors and conflict outcomes. Every other mention of that boundary — in the error-model registry ([10 — Error Model](./10-error-model.md#activity-end-states)), the edge-case rules ([12 — Edge Cases](./12-edge-cases.md)), and the interface **Errors** declarations — is an informative reference to this section; where any of them appears to draw a different line, this section governs ([01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership)).

**The rule: a structural schema violation is an Activity Error; a logical incompatibility is a conflict outcome.**

**Structural side — Activity Error.** An atom that fails the data model's own structural contract ([03 — Data Model](./03-data-model.md): identifier format, base fields, per-type field requirements) is **malformed input**:

- a missing or empty required field for the atom's type;
- an unrecognized `type` or `kind` value;
- a scalar out of range (e.g. `confidence`);
- an unregistered `Scope.tree`;
- an `id` that is malformed or whose `<type>` prefix disagrees with the `type` field;
- an `is_a` that does not resolve to a present-state atom — the definitional anchor; without it the atom cannot be interpreted at all.

A malformed proposal is rejected up front with an `extracting` / `schema_violation` Activity Error (the Primitive `cause` names the offending field) and never reaches the pipeline stages: nothing enters the snapshot, no outcome is emitted, and there is nothing to resolve.

**Logical side — conflict outcome.** Everything the pipeline classifies is a **conflict outcome**: the atom is well-formed and anchored, but it is incompatible with what it references or with what already exists —

- the anchor's category or its declared subject treatment (`WrongCategoryTypeConflict`, `InvalidSubject`);
- the referenced classification's or predicate's schema — required attributes, attribute value types, domain / range, `subject_kind` inheritance, characteristics vs. kind (stage 3);
- the logical model — disjointness, cardinality, asymmetry, irreflexivity;
- coexistence with other atoms — duplication, cycles, alias collisions;
- non-anchor reference health — an unresolved RelationAtom endpoint, referential `subject` reference, or `Scope.constraint_atoms` entry is admitted and flagged (`DeadEdge` / `DeadConstraint`), never rejected.

Each outcome carries a default mode and resolution paths; its effects are recorded per [Outcome recording](#outcome-recording). Outcomes are typed *results* of `propose`, never errors ([10 § Conflict-pipeline outcomes are results, not errors](./10-error-model.md#conflict-pipeline-outcomes-are-results-not-errors)).

**The boundary test.** Malformed input has exactly one fix — correct the input and resubmit — so it is an error. A violation of a *referenced definition's* rules, or of the atom's compatibility with other atoms, may equally be resolved in the store — re-anchoring, revising the referenced definition, retiring a counterpart — so it belongs to the review machinery and **MUST** surface as a conflict outcome, even where its default mode hard-blocks the proposal. An implementation **MUST NOT** move a check from one side of the fence to the other.

**The revision surface.** The same structural contract gates the direct revision surface as a `revising` / `schema_violation` Activity Error ([`interfaces/atoms.md § update`](../interfaces/atoms.md#update)). The revision gate additionally enforces the revised atom's **own** resolved classification / predicate schema: a revision is a single-atom mutation with no candidate to admit and flag, so an edit that would break the atom's own schema is a defective edit, rejected before mutation. Logical incompatibility with *other* atoms remains the pipeline's territory — the entailment recompute a revision triggers re-runs the T3 checks and surfaces conflict outcomes ([When the pipeline runs](#when-the-pipeline-runs)).

## Pipeline stages

A conformant implementation MUST execute these stages in order for any atom proposal. Stages MAY short-circuit when a hard-block outcome is reached.

| Stage | Checks | Outcomes it can emit |
|---|---|---|
| **1. Structural validation** | Field presence per type; `subject` treatment per classification's `subject_kind`; `is_a` target type per id-prefix rules (entity→classification, relation→predicate, …); resolution of `RelationAtom` endpoint, referential `subject`, and `Scope.constraint_atoms` references — an unresolved reference is admitted but flagged ([12 — Edge Cases § Dead AtomId references](./12-edge-cases.md#dead-atomid-references)) | `InvalidSubject`, `WrongCategoryTypeConflict`, `DeadEdge`, `DeadConstraint` |
| **2. Alias resolution** | Match proposed atom's prose subject against existing ClassificationAtoms' `aliases` (within-tree); resolve to canonical AtomId where possible; detect a proposed ClassificationAtom's `aliases` overlapping those of a different same-tree classification | `AliasCollision` (otherwise no outcome — internal normalization) |
| **3. Schema validation** | Walk `is_a` chain; collect cumulative `attributes` + `required_relations` + `optional_relations` + `disjoint_with` + `domain_classifications`; resolve `subject_kind`; for predicates, resolve the kind's hard characteristics and validate any `chain` declaration ([03 § Chain derivation](./03-data-model.md#chain-derivation): steps resolve to PredicateAtoms; the declaring kind is not composition-family); validate against the proposed atom's payload, attribute values, declared characteristics, and outgoing edges | `MissingRequiredAttribute`, `AttributeTypeViolation`, `MissingRequiredRelation`, `RelationCardinalityExceeded`, `DisjointGeneralizationConflict` (when classified under disjoint targets), `RelationDomainViolation` (RelationAtom source/target vs predicate domain/range), `SubjectClassificationViolation` (referential subject referent vs classification `domain_classifications`), `SubjectKindInheritanceConflict` (subject_kind vs ancestor), `CharacteristicKindConflict` (predicate characteristics or chain declaration vs its kind) |
| **4. Duplicate / similarity detection** | Within-tree: structural match against canonical atoms in the same tree | `Duplicate`, `Refines`, `Generalizes`, `Supersedes` |
| **5. Edge-specific structural checks** | For RelationAtoms: single-parent rule for `composition`; self-loop (`subject == object`) detection on irreflexive kinds; cycle detection in the subsumption hierarchy (`specialization` edges) and for other asymmetric kinds; direction-reversal detection for `specialization` edges; redundancy between the `is_a` field and an explicit `specialization` edge; equivalence-endpoint validity | `CompositionConflict`, `RelationKindConflict`, `IrreflexivityViolation`, `GeneralizationCycle`, `GeneralizationEquivalenceConflict`, `GeneralizationAliasConflict`, `DuplicateClassification`, `AsymmetryViolation`, `InvalidEquivalenceEndpoint` |
| **6. Equivalence / alias coherence** | Detect `kind=equivalence` between two ClassificationAtoms (suggests alias merge); detect same-tree `kind=equivalence` (suggests Duplicate); detect disjointness ↔ equivalence contradictions | `EquivalenceVsAlias`, `SameTreeEquivalenceConflict`, `DisjointEquivalenceConflict` |
| **7. Capacity check** | Count direct children of the target ClassificationAtom; emit if `max_children` exceeded | `CategoryMaxChildrenExceeded` |
| **8. Promotion suggestions** | A ClassificationAtom previously used only as a leaf acquires incoming `is_a` usage — suggest promotion semantics | `PromoteClassificationLeafToHub` |
| **9. Cross-tree similarity (advisory)** | High-similarity match against atoms in a different tree; never auto-merges | `CrossTreeEquivalenceCandidate` (soft-prompt; suggests `kind=equivalence` link) |

These nine stages run at **atom proposal**. The pipeline also runs on **entailment recomputation** (T3 — see [03 — Data Model](./03-data-model.md)): disjointness propagated through `is_a` chains, and `functional` / `inverse_functional` violations among **asserted** edges (resolved through the sub-predicate hierarchy; derived edges never count toward cardinality-like characteristics — [03 § Predicate kinds](./03-data-model.md#predicate-kinds)), emit `DisjointGeneralizationConflict`, `FunctionalViolation`, and `InverseFunctionalViolation` respectively (outside the numbered proposal stages).

The **[Outcome registry](#outcome-registry) is the authoritative definition** of every outcome — its trigger (meaning), default resolution mode, and required meta. The per-stage "Outcomes it can emit" column is a routing view (which stage detects what), not the definition.

## Resolution modes

Each conflict outcome declares its default resolution mode:

| Mode | Behavior |
|---|---|
| **auto-resolve** | The pipeline closes the situation itself by applying a deterministic resolution. The effects emit ordinary events carrying the outcome kind ([Outcome recording](#outcome-recording)); no human review required. |
| **soft-prompt** | The pipeline records the situation in atom-space with default resolution recommendation(s) for the human reviewer ([Outcome recording](#outcome-recording)). The atom proposal continues; resolution may be deferred. |
| **hard-block** | The pipeline rejects the proposal: the candidate is not added to the snapshot and is returned in full in the `propose` result. The human reviewer MUST resolve the cause before the proposal can be re-submitted. |

Implementations MAY treat soft-prompts as hard-blocks for strict deployments. They MUST NOT downgrade hard-blocks to soft-prompts.

A hard-block leaves the atom **never committed** (not added) — there is nothing to tombstone; the rejection is recorded only as a `Rejected` audit event carrying the candidate ([Outcome recording](#outcome-recording)). Likewise, a removal outcome that drops a never-committed proposal (e.g., a proposal-time `Duplicate`, where the *new* atom never entered) MAY auto-resolve and creates no tombstone. Only the removal of a *previously committed* atom is tombstoned (two-step + GC; see [07 — Atom Lifecycle](./07-atom-lifecycle.md)).

## Outcome registry

The registry's entries — every outcome kind with its trigger (meaning), default resolution mode, and required meta — are enumerated in [Appendix A § Conflict outcomes](./appendix-a-registries.md#conflict-outcomes). The set is exhaustive: the pipeline MUST NOT emit an outcome kind outside it. The same set is mirrored as the `ConflictOutcomeKind` enum in [`interfaces/atoms.md`](../interfaces/atoms.md); the appendix enumeration and the enum stay in lockstep per the mirror rule in [Appendix A — Registries](./appendix-a-registries.md).

## Revision and succession

Whether proposed content *revises* an existing canonical atom or *replaces it with a different claim* is decided by the provenance-fidelity rule ([03 — Data Model § Mutability and revision](./03-data-model.md#mutability-and-revision)): content that remains an honest record of the existing atom's own provenance is a revision; content that needs new, contrary provenance is a different claim.

**Revision — apply in place.** When stage 4 matches a proposal to an existing canonical atom as the same claim (`Duplicate`) and the proposed content is a corrected or updated reading of that atom, the resolution applies the content to the existing atom as an in-place update: the atom's `id` is unchanged, the proposal's `Source` is appended, an `Updated` event is emitted, and no new atom enters. These are the pipeline's **revision-class** resolutions; `propose` reports the revised atoms in its result ([`interfaces/atoms.md § propose`](../interfaces/atoms.md#propose)). The same mutation is available directly, without pipeline involvement, via [`Atoms.update`](../interfaces/atoms.md#update).

**Succession — `Supersedes`, gated.** A proposal that replaces an existing canonical atom with a *different claim* is classified `Supersedes` only when the new claim **fully absorbs the old one as a compatible refinement** — every reading the old atom supported remains supported by the new one. Resolution: the new atom enters `active`; the prior atom transitions to `superseded` with `meta.superseded_by`; asserted references to it then resolve to the successor ([07 § Observable resolution invariant](./07-atom-lifecycle.md#observable-resolution-invariant)). The gate binds in both directions: content that passes the provenance-fidelity rule is a revision, never a supersession; and a **contrary** replacement MUST NOT be classified `Supersedes` — the new claim enters as a new atom and the prior one is phased out through review (`deprecated`, then a removal outcome), so its dependents re-validate ([07 § Revision, succession, and the prior atom's status](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status)).

A proposal that is a more specific or more general statement of an existing claim — rather than a reading of the same claim or a replacement — remains `Refines` / `Generalizes`: both atoms are kept and linked.

Resolving a `hanging` atom by revision (Adapt) composes these mechanisms with an explicit status transition; the path is defined in [07 § Adapt](./07-atom-lifecycle.md#adapt--special-case).

## Aliases vs. `kind=equivalence` vs. `Duplicate` — bright-line rules

Three mechanisms can express "these are the same thing." They operate at different levels and are not interchangeable. The Conflict pipeline routes between them per the following rules:

| Mechanism | Level | What it links | When to use |
|---|---|---|---|
| `aliases` (on ClassificationAtom / PredicateAtom schema) | Names | strings → strings | Surface-form variations of the same class or predicate name. Used by extraction to match incoming text. |
| `RelationAtom` whose PredicateAtom is_a `kind=equivalence` | Atoms | AtomId → AtomId | Two distinct atoms in **different trees** making the same claim; both are kept and linked. Equivalence is **cross-tree only** — a same-tree equivalence is invalid and is caught as `SameTreeEquivalenceConflict`, which reduces to `Duplicate`. |
| `Duplicate` conflict outcome | Ingestion (same-tree, same-claim) | new atom → existing canonical atom | At ingestion in the same tree, the new atom is detected as the same; the duplicate is removed or tombstoned. One atom survives. |

One-line rules:
- "Same concept, different names" → aliases.
- "Same claim, **different trees** (keep both, linked)" → `kind=equivalence`.
- "Same claim, same tree, one survives" → `Duplicate`.

Aliases and `kind=equivalence` rarely overlap because aliases require a ClassificationAtom or PredicateAtom subject and operate on strings; `kind=equivalence` operates on AtomIds across any atom type except RelationAtom (which is excluded — see below).

## `kind=equivalence` exclusions

A `RelationAtom` with PredicateKind `equivalence` MUST NOT have a RelationAtom as either `subject` or `object`. The Conflict pipeline rejects malformed equivalence at proposal time with `InvalidEquivalenceEndpoint` (hard-block). Reason: if two RelationAtoms' underlying endpoints are equivalent, the RelationAtoms are equivalent by extension; declaring it explicitly carries no information.

## Tree-bounded scope

`Duplicate` detection runs **within-tree only** (per `Scope.tree`). Cross-tree similarity is advisory — surfaced as `CrossTreeEquivalenceCandidate` and never auto-merged.

Disjointness checks run **within-tree only**. Two ClassificationAtoms declared disjoint in `planning` may be equivalent or generalized in `implementation`.

## Confidence thresholds

| Threshold | Default | Where applied |
|---|---|---|
| `Duplicate` auto-resolve threshold | 0.95 | `Duplicate` outcome with structural match confidence ≥ this auto-merges; below this becomes soft-prompt |
| `composition` cascade threshold | 0.8 | Composition cascade (source removed → target hanging) fires only when the composition RelationAtom's `confidence` ≥ this |

Both are deployment-overridable. Conformance requires *some* threshold; a specific default is recommended but not mandated.

## Outcome recording

A conflict outcome has no identifier, no store of its own, and no parallel review queue. The `ConflictOutcomeKind` vocabulary — the [Outcome registry](#outcome-registry) — classifies; atoms and their statuses carry the state. Every effect the pipeline produces MUST be emitted as an ordinary Atoms event with the outcome kind in its payload (`Added` / `Updated` / `StatusChanged` / the removal-outcome events — payload shapes in [`interfaces/atoms.md § changes_since`](../interfaces/atoms.md#changes_since)); there are no dedicated conflict-outcome event kinds. A status transition the pipeline drives MUST additionally record the classification in the atom's `status.meta` — `outcome_kind`, `outcome_stage`, `outcome_confidence` ([`StatusMeta`](../interfaces/00-shared-types.md)).

### What each mode records

| Mode | Persistent record | Events |
|---|---|---|
| **auto-resolve** | The applied effects themselves — added or revised atoms, status transitions (classification in `status.meta`). | The effects' ordinary events, each carrying `outcome_kind`; a resolution that drops a never-committed proposal (e.g., a high-confidence `Duplicate` fold) emits the `Rejected` audit event carrying the candidate |
| **soft-prompt** | The proposal enters the snapshot (asserted). The pending situation is recorded under the reserved `aala.conflict` annotation: on a `conflicts_with` **linkage RelationAtom** when the situation relates two or more persisted atoms ([The linkage atom](#the-linkage-atom)), on the affected atom itself when it involves one. | The linkage atom's `Added` (carrying `outcome_kind`), or the affected atom's `Updated` |
| **hard-block** | None. The candidate is never committed and leaves no tombstone; it is returned in full in the `propose` result ([`interfaces/atoms.md § propose`](../interfaces/atoms.md#propose)). | One `Rejected` audit event carrying the candidate and the `outcome_kind`; it references no committed atom and applies as a **no-op** to materialized state ([05 § Causal ordering](./05-delta-streams.md#causal-ordering-within-a-stream)) |

The annotation value is a list of classification records — `ConflictRecord[]`, the same shape `propose` returns ([`interfaces/atoms.md`](../interfaces/atoms.md#propose)); an atom or linkage MAY carry several pending records. The `aala.conflict` key is reserved for this use.

### The linkage atom

A pending situation relating two or more persisted atoms is materialized as a RelationAtom whose predicate is the standard-library **`conflicts_with`** ([03 § Standard library](./03-data-model.md#standard-library--pre-registered-atoms-per-tree)): `association` kind, symmetric (self-`inverse_of`; T1 materializes the reverse edge), no cascade, no hard characteristics. Its `subject` is the primary affected atom (the proposal, for proposal-time detections) and its `object` the principal counterpart; further involved atoms are listed in its `aala.conflict` records. The linkage is an ordinary asserted atom — a claim about claims: it enters `active`, appears in the snapshot's reviewable diff, and is queryable, traversable, and removable like any other atom.

### Pending review

"What needs review" is answered entirely through the existing read and navigation surfaces — there is no separate review API:

- traverse `conflicts_with` edges ([`traverse_relations`](../interfaces/atoms.md#traverse_relations) filtered to the predicate);
- list atoms carrying the `aala.conflict` annotation in a scope or tree ([`list_scope`](../interfaces/atoms.md#list_scope), [`list_by_tree`](../interfaces/atoms.md#list_by_tree));
- filter listings by status (`hanging`, `deferred`) and inspect `status.meta` for the driving classification;
- navigate scope structure to localize the affected areas ([Hierarchical Navigation](../interfaces/hierarchical-nav.md)).

### Resolution

Resolution is performed with the ordinary mutation surfaces, with rationale — there is no resolve-an-outcome method:

- a status transition — [`update_status(atom_id, outcome, rationale, meta?)`](../interfaces/atoms.md#update_status);
- an in-place revision — [`update(atom_id, changes, rationale)`](../interfaces/atoms.md#update);
- a subsequent ingestion ([`propose`](../interfaces/atoms.md#propose)) that revises or replaces the disputed claims.

A resolution that disposes of a pending situation MUST also clear its record: retire the linkage RelationAtom (an ordinary `removed` transition carrying the resolution rationale) or revise the `aala.conflict` annotation away. Idempotency and concurrency are the mutating method's own declared contracts — resolution adds no rules of its own.

The Delta Stream is the audit trail: the events that recorded a situation and the events that resolved it are ordinary stream events, retained regardless of what later happens to the atoms involved ([07 § Garbage collection](./07-atom-lifecycle.md#garbage-collection)).
