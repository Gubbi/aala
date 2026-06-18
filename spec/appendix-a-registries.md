# Appendix A — Registries

This appendix is **normative**. It is the single home of the specification's closed registries ([01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership)): each enumeration below is exhaustive, and no chapter re-enumerates it. The appendix carries **enumeration only** — each section names the **owning chapter**, which defines what the entries mean, the rules they obey, and what extending the registry costs ([11 — Versioning](./11-versioning.md)). Where any other text appears to enumerate a registry differently, this appendix governs.

Event unions and type shapes are not registries; they live in [`interfaces/`](../interfaces/) per the normative-ownership rule. Where an interface type mirrors a registry here (e.g. `ConflictOutcomeKind`), the section says so, and the two MUST stay in lockstep.

## Containers

Semantics and conformance obligations: [02 — Conformance § Required containers](./02-conformance.md#required-containers).

| Container | Status |
|---|---|
| [Ingestion](../interfaces/ingestion.md) | REQUIRED |
| [Atoms](../interfaces/atoms.md) | REQUIRED |
| [Orchestration](../interfaces/orchestration.md) | REQUIRED |
| [LLM Gateway](../interfaces/llm-gateway.md) | REQUIRED |
| [Hierarchical Navigation](../interfaces/hierarchical-nav.md) | OPTIONAL |
| [Blast Radius](../interfaces/blast-radius.md) | OPTIONAL |
| [Quality](../interfaces/quality.md) | OPTIONAL |

[Projection](../interfaces/projection.md) is a cross-cutting **read facet**, not a container, and is never a registry entry; its obligations are in [02 — Conformance § Required facet — Projection](./02-conformance.md#required-facet--projection).

## Identifier minting

Identifier **formats** are type shapes owned by [`interfaces/00-shared-types.md § Identifiers`](../interfaces/00-shared-types.md#identifiers). This registry is the normative home for the rest of an identifier's contract, fixed per identifier type: the **minting authority** (who creates it), the **uniqueness scope** (the boundary within which it names exactly one referent), and the **stability** of the name. One rule closes every row: within its uniqueness scope an identifier names exactly one referent, forever — an implementation **MUST NOT** re-mint an identifier for a different referent, even after the original referent is deleted, discarded, or purged. A uniqueness scope of *deployment* means one logical deployment ([02 — Conformance § Single-deployment scope](./02-conformance.md#single-deployment-scope)).

| Identifier | Minted by | Uniqueness scope | Stability |
|---|---|---|---|
| `AtomId` | **Atoms** — at admission of a new asserted atom ([`propose`](../interfaces/atoms.md#propose)); the entailment engine mints the ids of the derived atoms it materializes | deployment — unique within every snapshot; the same id in two snapshots is the same logical atom ([04 § Atom identity across snapshots](./04-snapshots.md#atom-identity-across-snapshots)) | permanent — held through every in-place revision (`Updated`) and status transition; a removal-outcome tombstone retains it |
| `FragmentId` | **Ingestion** — [`accept`](../interfaces/ingestion.md#accept), on acceptance only; a rejected fragment is never persisted and receives no id | deployment | resolvable via `Ingestion.get` until the fragment is deleted under deployment retention policy |
| `SnapshotId` | **Orchestration** — [`fork`](../interfaces/orchestration.md#fork) | deployment | permanent — a snapshot keeps its id through every state, including `discarded` |
| `Ref` | the **emitting container** — one per event it delivers, real Delta-Stream events and `as_stream` bootstrap events alike ([05 — Delta Streams](./05-delta-streams.md)) | per stream — one container's Delta Stream within one snapshot; a ref that does not name a checkpoint of the bound snapshot (foreign, discarded, or unrecognized) fails with `stale_ref` ([05 § Cross-snapshot behavior](./05-delta-streams.md#cross-snapshot-behavior)) | names its checkpoint for the snapshot's life ([05 § Stream retention](./05-delta-streams.md#stream-retention)); resumability ends when the snapshot is discarded. Opaque — equality is the only client-side comparison; ordering is held by the producer ([05 § Ordering](./05-delta-streams.md#ordering)) |
| `IngestId` | **Ingestion** — [`accept`](../interfaces/ingestion.md#accept), one per evaluation: an idempotent replay returns the originally minted id; each rejecting evaluation mints a fresh one | deployment — names exactly one ingest operation | permanent — the correlation audit trail ([05 § Correlation across streams](./05-delta-streams.md#correlation-across-streams)) outlives the fragment |
| `RequestId` | **Orchestration (the perimeter)** — at admission of each externally initiated request other than `ingest`, reads included ([orchestration § Request entry points](../interfaces/orchestration.md#request-entry-points)); a write stamps it on downstream change events, a read carries it only in errors and telemetry | deployment — names exactly one request | permanent — correlation audit trail |
| `SystemOpId` | the **container initiating an internal operation** — rebuild, integrity sweep, maintenance pass ([05 § Correlation across streams](./05-delta-streams.md#correlation-across-streams)) | deployment — names exactly one internal operation | permanent — correlation audit trail |
| `BlastId` | **Blast Radius** — [`analyze`](../interfaces/blast-radius.md#analyze), when it opens a report; a replay against unchanged atom state returns the existing open report's id ([08 § Determinism and report identity](./08-blast-radius.md#determinism-and-report-identity)) | deployment — names exactly one report | resolvable while the report is retained ([08 § Storage and retention](./08-blast-radius.md#storage-and-retention)); applied resolutions survive on the atoms after the report is collected |
| `TreeId` | the **caller** of [`register_tree`](../interfaces/orchestration.md#register_tree) — deployer-chosen; registration validates format (`invalid_tree`) and collision (`already_registered`) | deployment — the tree registry admits each id once | immutable — no operation rewrites it; every `Scope.tree` reference relies on that ([12 § Tree lifecycle](./12-edge-cases.md#tree-lifecycle)) |
| `CallId` | **LLM Gateway** — one per call it executes, keying the call's structured observability record ([llm-gateway § Notes for implementers](../interfaces/llm-gateway.md#notes-for-implementers)) | deployment — names exactly one gateway call | resolvable while the call record is retained; the key that feedback and failed-query records carry ([`Quality.record_feedback`](../interfaces/quality.md#record_feedback)) |
| `message_id` | the **source adapter**, outside this contract, before the fragment reaches [`accept`](../interfaces/ingestion.md#accept) | deployment — the adapter **MUST** mint ids unique across *all source systems* feeding the deployment; a per-source namespace prefix is **RECOMMENDED** (mirroring `ContentKind`'s dot-namespace convention). Source-native ids (a chat timestamp, a PR comment number) are unique only within their own system; un-namespaced, a cross-system collision silently replays an unrelated fragment | stable re-presentation — the same logical source fragment re-presented **MUST** carry the same `message_id`; that stability is what makes it the idempotency key for `accept` and `propose` ([shared-types § Idempotency](../interfaces/00-shared-types.md#idempotency)) |

Three notes close the registry:

- `CorrelationId` is not separately minted — it is whichever of `IngestId` / `RequestId` / `SystemOpId` the initiating context stamped ([05 § Correlation across streams](./05-delta-streams.md#correlation-across-streams)).
- `Cursor` continuation tokens are minted per page by the serving implementation; their validity scope and lifetime are owned by [shared-types § Paged collection reads](../interfaces/00-shared-types.md#paged-collection-reads) and they are not registry entries.
- Record-local keys (`GoldenSetId`, `case_id`, `BenchmarkVersion`) and path-shaped names (`SectionPath`, `ScopePath`, `ContentKind`, `UseCaseKey`, `ProfileKey`) are names, not minted identifiers: caller- or deployer-chosen, scoped to the surface that declares them.

## Predicate kinds

Semantics — characteristic enforcement, inverse pairing, the composition family's non-transitivity, validation rules: [03 — Data Model § Predicate kinds](./03-data-model.md#predicate-kinds). Edges are written subject → object.

| Kind | Edge meaning (subject → object) | Hard characteristics | Cascade | `inverse_of` |
|---|---|---|---|---|
| `association` | generic link | none | none | `association` |
| `equivalence` | same claim | symmetric, transitive | none | `equivalence` |
| `specialization` | subject is-a-kind-of object (Cat → Animal) | asymmetric, irreflexive, transitive | none | `generalization` |
| `generalization` | subject has-subtype object (Animal → Cat) | asymmetric, irreflexive, transitive | none | `specialization` |
| `aggregation` | subject (whole) aggregates object (part); shared parts allowed | asymmetric, irreflexive, transitive | none | `member_of` |
| `member_of` | subject (part) is a member of object (whole) | asymmetric, irreflexive, transitive | none | `aggregation` |
| `composition` | subject (whole) composed-of object (part); **direct**, exclusive part | asymmetric, irreflexive, inverse_functional | source (whole) enters any removal outcome → target (part) `hanging`, gated by confidence ≥ threshold (default 0.8) | `component_of` |
| `component_of` | subject (part) component-of object (whole); **direct**, exclusive | asymmetric, irreflexive, functional | target (whole) enters any removal outcome → source (part) `hanging`, gated by confidence ≥ threshold | `composition` |
| `dependency` | subject (dependent) depends-on object (prerequisite) | asymmetric, irreflexive, transitive | target (prerequisite) leaves `active` (any transition to `hanging`, `deferred`, `deprecated`, or a removal outcome) → source (dependent) `hanging` | `enablement` |
| `enablement` | subject (prerequisite) enables object (dependent) | asymmetric, irreflexive, transitive | source (prerequisite) leaves `active` → target (dependent) `hanging` | `dependency` |

The **Cascade** column summarizes the per-kind triggers informatively. The normative definition of impact propagation — trigger application, channel ordering, fixpoint, confidence gating, and emissions — is [07 — Atom Lifecycle § Cascade rules](./07-atom-lifecycle.md#cascade-rules).

## Standard library

Semantics: [03 — Data Model § Standard library](./03-data-model.md#standard-library--pre-registered-atoms-per-tree); the per-tree pre-registration obligation is [02 — Conformance § Standard-library registry](./02-conformance.md#standard-library-registry). The standard library comprises the ten PredicateKindAtoms — one per kind in [§ Predicate kinds](#predicate-kinds), each a flat root (no `is_a`) carrying its `inverse_of` pointer — plus the classifications and predicates below.

### Classifications

| Pre-registered classification | `is_a` (parent) | `subject_kind` | Purpose |
|---|---|---|---|
| `Classification("Thing")` | — (root) | abstract | Single entity-domain root; its `max_children` ceiling forces grouping rather than flat roots. |
| `Classification("Claim")` | `Thing` | referential | Groups referential claim classes. |
| `Classification("Named-Individual")` | `Thing` | named | Groups named-individual classes. |
| `Classification("Decision")` | `Claim` | referential (inherited) | Decision claims. |
| `Classification("Constraint")` | `Claim` | referential | Constraint claims; ConstraintAtoms gate other atoms via `Scope.constraint_atoms`. |
| `Classification("Behavior")` | `Claim` | referential | Behavior claims. |
| `Classification("Capability")` | `Claim` | referential | Capability claims. |
| `Classification("Scenario")` | `Claim` | referential | Scenario flows. |
| `Classification("Concept")` | `Named-Individual` | named | Named domain concepts. |
| `Classification("Person")` | `Named-Individual` | named | Named individuals — people. |
| `Classification("Organization")` | `Named-Individual` | named | Companies, teams. |
| `Classification("Place")` | `Named-Individual` | named | Locations. |
| `Classification("Time")` | `Named-Individual` | named | Time-anchored entities. |

### Predicates

| Pre-registered predicate | `is_a` (kind) | `inverse_of` | `chain` | Purpose |
|---|---|---|---|---|
| `refines` | `specialization` | `generalizes` | — | Claim refinement (specific → general); an asserted `refines` edge materializes its `generalizes` inverse via T1. |
| `generalizes` | `generalization` | `refines` | — | Inverse of `refines`. |
| `component_of` | `component_of` | `has_component` | — | Direct, exclusive parthood (part → whole). The *predicate* is distinct from the same-named *kind* its `is_a` bottoms out at. |
| `has_component` | `composition` | `component_of` | — | Inverse of the `component_of` predicate (whole → part). |
| `part_of` | `member_of` | `has_part` | `component_of ∘ component_of+` | Indirect containment (part → whole): derived for every path of two or more asserted `component_of` edges ([03 § Chain derivation](./03-data-model.md#chain-derivation)); T1 materializes the `has_part` inverses. Direct parthood remains the asserted `component_of` edge. |
| `has_part` | `aggregation` | `part_of` | — | Inverse of `part_of` (whole → part). |
| `conflicts_with` | `association` | `conflicts_with` (self) | — | Conflict linkage — records a pending conflict situation between persisted atoms; symmetric (T1 materializes the reverse edge), no cascade. Created by the Conflict pipeline, carrying the classification in its `aala.conflict` annotation ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)). |

## Conflict outcomes

Semantics — pipeline stages, resolution modes, recording, the schema-violation fence: [06 — Conflict Classification § Outcome registry](./06-conflict-classification.md#outcome-registry). The set is mirrored as the `ConflictOutcomeKind` enum in [`interfaces/atoms.md`](../interfaces/atoms.md).

| Outcome | Triggered by | Default mode | Required meta |
|---|---|---|---|
| `Duplicate` | Same claim, same tree (high structural match against an existing canonical atom). When the proposal carries a corrected or updated reading, resolution applies it to the canonical atom as an in-place **revision** ([06 § Revision and succession](./06-conflict-classification.md#revision-and-succession)); otherwise the proposal is dropped and its `Source` MAY be appended to the canonical atom | soft-prompt (auto-resolve when confidence ≥ 0.95) | `meta.duplicate_of` |
| `Refines` | New atom is a **more specific** version of an existing canonical atom (e.g., a constraint that narrows a more general one). Both atoms are kept (no status change); resolution materializes a `refines` RelationAtom (new → existing). | soft-prompt | `meta.refines` (the existing, more-general atom) |
| `Generalizes` | New atom is a **more general** version of an existing canonical atom — the symmetric case of `Refines`. Both atoms are kept (no status change); resolution materializes a `refines` RelationAtom (existing → new), i.e. a `generalizes` edge (new → existing). | soft-prompt | `meta.generalizes` (the existing, more-specific atom) |
| `Supersedes` | New atom is a **different claim** that replaces existing canonical atom(s) and fully absorbs them as a compatible refinement — the succession gate ([06 § Revision and succession](./06-conflict-classification.md#revision-and-succession); [07 § The succession gate](./07-atom-lifecycle.md#the-succession-gate)). Detected from an explicit successor declaration on the proposal or a structural replacement pattern | soft-prompt | `meta.superseded_by` (on the prior atom) |
| `CompositionConflict` | A `RelationAtom` whose PredicateKind is `composition` is proposed and its `object` is already the target of another active **asserted** composition edge (single-parent rule; derived edges never count — [03 § Predicate kinds](./03-data-model.md#predicate-kinds)) | hard-block | both composition edges |
| `RelationKindConflict` | Two RelationAtoms share `(subject, object, scope)` but reference PredicateAtoms with different kinds | soft-prompt | both relations |
| `GeneralizationCycle` | The subsumption hierarchy closes a directed cycle: any cycle that runs through at least one `is_a` field link (depth ≥ 2 — an `is_a` 2-cycle and mixed `is_a`/edge 2-cycles included), or a cycle of depth ≥ 3 through explicit `kind=specialization` edges alone. (A 2-cycle of explicit `kind=specialization` edges alone is `GeneralizationEquivalenceConflict`.) Detection is complete at every depth ([12 — Edge Cases § Cycles in `is_a`](./12-edge-cases.md#cycles-in-is_a)) | hard-block | the closing edge and the loop |
| `GeneralizationEquivalenceConflict` | Two `kind=specialization` edges exist with `(subject=A, object=B)` and `(subject=B, object=A)` in overlapping scope (2-cycle) | soft-prompt; resolution paths: promote to aliases, promote to `kind=equivalence`, pick a single direction, reject both | both edges |
| `GeneralizationAliasConflict` | A `kind=specialization` edge says "X is a kind of Y" while a ClassificationAtom registers X in Y's `aliases` (or the reverse) | soft-prompt | the edge and the alias graph entry |
| `WrongCategoryTypeConflict` | An atom's `is_a` target is not the atom type required for the atom's own `type` (e.g., an `entity` whose `is_a` does not point to a classification, or a `relation` whose `is_a` does not point to a predicate) | hard-block | the atom and the mistyped `is_a` target |
| `CategoryMaxChildrenExceeded` | A ClassificationAtom's direct-children count (sub-categories + classified instance atoms) exceeds `max_children` | soft-prompt | the saturated category |
| `EquivalenceVsAlias` | A `kind=equivalence` RelationAtom links two ClassificationAtoms; suggests collapsing into one via the aliases mechanism | soft-prompt | both classifications |
| `SameTreeEquivalenceConflict` | A `kind=equivalence` RelationAtom links two atoms in the **same** tree (equivalence is cross-tree only) | soft-prompt; reduces to `Duplicate` — one atom survives and the loser's surface form becomes an alias on the survivor; the same-tree equivalence edge is not kept | both atoms |
| `DisjointEquivalenceConflict` | Two atoms are declared both `disjoint_with` each other AND `kind=equivalence` linked | hard-block | both atoms and the equivalence edge |
| `DisjointGeneralizationConflict` | An atom is classified (via `is_a` or `kind=specialization`) under two ClassificationAtoms declared `disjoint_with` each other | hard-block | the atom and the disjoint classifications |
| `DuplicateClassification` | An atom has both `is_a = X` and a `kind=specialization` RelationAtom pointing at the same X | soft-prompt | the atom and the redundant edge |
| `PromoteClassificationLeafToHub` | A ClassificationAtom that previously had no incoming `is_a` acquires its first incoming `is_a` (suggests it's becoming a hub, not a leaf) | soft-prompt | the classification |
| `MissingRequiredAttribute` | An atom is classified under a ClassificationAtom whose cumulative `attributes` (walked via is_a chain) include a `required: true` slot the atom's `attributes` map does not provide | soft-prompt | the atom, the missing attribute key(s) |
| `AttributeTypeViolation` | An atom provides an `attributes` value whose type does not match the slot's declared `value_type` | hard-block | the atom, the attribute key, expected vs. actual type |
| `MissingRequiredRelation` | An atom is classified under a ClassificationAtom whose cumulative `required_relations` (walked via is_a chain) aren't satisfied by the atom's outgoing RelationAtoms (the `min` is not met) | soft-prompt | the atom, the missing relation constraint |
| `RelationCardinalityExceeded` | An atom has more outgoing RelationAtoms for a `(predicate, target_classification)` constraint than the constraint's `max` (`required_relations` or `optional_relations`) | soft-prompt | the atom, the relation constraint, the offending edges |
| `RelationDomainViolation` | A RelationAtom's source or target classification does not match its PredicateAtom's `domain_classifications` / `range_classifications` | hard-block | the relation, the predicate, the offending endpoint |
| `SubjectClassificationViolation` | A referential atom's `subject` reference points at an atom whose classification is not in the classification's `domain_classifications` | hard-block | the atom, the subject reference, the allowed classifications |
| `InvalidSubject` | An atom's `subject` violates the type's subject treatment rule (e.g., a `referential` classification's instance with prose subject) | hard-block | the atom |
| `CrossTreeEquivalenceCandidate` | High structural similarity to an atom in a different tree; never auto-merges | soft-prompt; recommends `kind=equivalence` link | both atoms |
| `FunctionalViolation` | T3 entailment detects two **asserted** RelationAtoms (resolved through the sub-predicate hierarchy) that violate a PredicateAtom's `functional` characteristic | hard-block | both relations, the predicate |
| `InverseFunctionalViolation` | T3 entailment detects two incoming **asserted** RelationAtoms (resolved through the sub-predicate hierarchy) that violate `inverse_functional` (other than composition's single-parent, which is caught by `CompositionConflict`) | hard-block | both relations, the predicate |
| `AsymmetryViolation` | RelationAtoms of an asymmetric kind (`aggregation` / `member_of` / `composition` / `component_of` / `dependency` / `enablement`) form a directed cycle of asserted edges — detected directly or through the entailment closure — violating the kind's hard `asymmetric` characteristic. (Subsumption cycles — `specialization` / `generalization` — surface as `GeneralizationCycle` / `GeneralizationEquivalenceConflict` instead.) | hard-block | the cycle edges, the kind |
| `InvalidEquivalenceEndpoint` | A `kind=equivalence` RelationAtom has a RelationAtom as its `subject` or `object` (forbidden — [06 § `kind=equivalence` exclusions](./06-conflict-classification.md#kindequivalence-exclusions)) | hard-block | the malformed equivalence relation |
| `IrreflexivityViolation` | An asserted RelationAtom has `subject == object` (directly, or via a cycle of asserted edges surfacing as a self-loop in the entailment closure) on a kind whose `irreflexive` characteristic is hard true | hard-block | the relation, the kind |
| `CharacteristicKindConflict` | A PredicateAtom's declared `characteristics` — or its `chain` declaration ([03 § Chain derivation](./03-data-model.md#chain-derivation)) — contradict the hard semantics of its `predicate_kind` (resolved by walking the is_a chain to the kind), e.g. a chain declared on a composition-family predicate | hard-block | the predicate, the conflicting characteristic or chain, the kind |
| `SubjectKindInheritanceConflict` | A ClassificationAtom's `schema.subject_kind` contradicts an ancestor that already fixed it (walked via the is_a chain) | hard-block | the classification, the ancestor that fixed `subject_kind` |
| `DeadEdge` | A proposed RelationAtom's `subject` or `object` — or a referential atom's `subject` reference — names an AtomId absent from the current snapshot. The atom enters `active` but the reference is dead: it transmits no cascade, and referent-dependent validation is held with the flag; materialization clears the record, runs the held validation, and evaluates cascade triggers against the referent's current state ([12 — Edge Cases § Dead AtomId references](./12-edge-cases.md#dead-atomid-references)) | soft-prompt | the referring atom, the unresolved reference id(s) |
| `DeadConstraint` | A proposed atom's `Scope.constraint_atoms` entry references an AtomId absent from the current snapshot. The atom enters; the scope-premise cascade ignores the dead entry until the referent materializes ([12 — Edge Cases § Dead AtomId references](./12-edge-cases.md#dead-atomid-references)) | soft-prompt | the atom, the unresolved constraint id(s) |
| `AliasCollision` | A proposed ClassificationAtom's `aliases` overlap the aliases of a **different** ClassificationAtom in the same tree (detected during alias resolution, stage 2). The earlier registration keeps the alias; the proposal is flagged for review | soft-prompt | both classifications, the colliding alias string(s) |

## Use-case keys

Semantics — routing, schema validation, registration rules: [09 — LLM Gateway § Standard registered use-case keys](./09-llm-gateway.md#standard-registered-use-case-keys).

| Key | Intent | Produces |
|---|---|---|
| `aala.extraction` | Extract atoms from a `NormalizedFragment`. Producer must classify each extracted atom under a registered ClassificationAtom and emit RelationAtoms with valid PredicateAtom references. | structured (matches atom schemas) |
| `aala.conflict_judge` | Classify a proposed atom against an existing canonical candidate. Returns one of the conflict outcomes in [§ Conflict outcomes](#conflict-outcomes). | structured |
| `aala.classification_suggest` | Suggest a `ClassificationAtom` (and `is_a` chain) for a new atom whose classification isn't explicit in the source. | structured |
| `aala.predicate_suggest` | Suggest a `PredicateAtom` (and the underlying `PredicateKindAtom`) for a relationship surface form found in text. | structured |
| `aala.blast_implicit` | "Does atom Y still hold under the new premise from decision X?" Used by Blast Radius's optional implicit-pass. | structured |
| `aala.projection_narrative` | Render connective prose anchored to the previous projection. Called inside a contentful container's projection facet (Atoms primarily). | freeform |
| `aala.label_synthesis` | Generate a semantic label + optional description for a navigation tree node or a derived classification. | freeform |
| `aala.faithfulness_judge` | Judge prose against a reference set of atoms — missed claims, distorted claims, unsupported additions. Behind [`Quality.verify_faithfulness`](../interfaces/quality.md#verify_faithfulness) and faithfulness golden cases. | structured |
| `aala.relevance_filter` | Classify a fragment as relevant / not-relevant for downstream extraction. | structured (binary or scored) |
| `aala.embedding_default` | Default text-to-vector embedding model. | embedding |

## Projection output profiles

Semantics — the single-profile-per-deployment rule, the default, the re-projection cost of changing it: the [Projection facet contract § Output profile](../interfaces/projection.md#output-profile). The default profile, `okf`, is defined normatively in [Appendix B — OKF Profile](./appendix-b-okf-profile.md). The materialized projection store conforms to exactly one profile, advertised as the `ProfileKey` in [`OrchestrationCapabilities.projection_profile`](../interfaces/orchestration.md#capabilities).

**Profile contract.** Any output profile **MUST** define all of:

- **Document structure** — the frontmatter schema (the set and meaning of structured-head keys), the body section conventions (which headings carry which content), and the link and citation conventions.
- **Content mapping** — how aala state renders into a document: a document's primary subject's classification → the document `type`; the atom label → `title`; the summary → `description`; `provenance` → citations; `status.changed_at` → the document `timestamp`; RelationAtoms → inter-document links.
- **Index and log serialization** — how [`ProjectionIndex`](../interfaces/projection.md#read_index) materializes (the index document) and how the facet's Delta Stream ([`changes_since`](../interfaces/projection.md#changes_since)) materializes (the chronological log).
- **Deterministic, machine-parseable frontmatter** — the structured head a caller reads via [`ProjectionDoc.frontmatter`](../interfaces/projection.md#read) without re-parsing the body, byte-stable under the determinism tuple ([Projection § Determinism](../interfaces/projection.md#determinism)).

| `ProfileKey` | Status | Defined in |
|---|---|---|
| `okf` | **default** | [Appendix B — OKF Profile](./appendix-b-okf-profile.md) |

A deployment configures **exactly one** profile — `okf` unless overridden — and **MAY** register one custom profile that meets the profile contract above. There is no per-call profile selection; changing the configured profile requires a full re-projection ([Projection § Notes for implementers](../interfaces/projection.md#notes-for-implementers)).

## Activities (Tier 2)

Semantics: [10 — Error Model § The standardized Tier-2 vocabulary](./10-error-model.md#the-standardized-tier-2-vocabulary). The set is mirrored as the `Activity` type in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md#errors--the-three-tier-model).

| `activity` | The work that failed | Raised chiefly by |
|---|---|---|
| `ingesting` | Accepting and normalizing a raw fragment | Ingestion |
| `extracting` | Turning a fragment into candidate atoms | Atoms |
| `classifying_conflicts` | Running the conflict pipeline on a proposal | Atoms |
| `transitioning` | Applying a status change + its cascade | Atoms |
| `revising` | Applying an in-place content revision to an atom | Atoms |
| `snapshotting` | Forking / publishing / discarding a snapshot, or rebinding a session's snapshot | Orchestration, Atoms |
| `managing` | Administrative registry work — registering / renaming a tree, capability config | Orchestration |
| `querying` | Reading atoms / projected prose / indexes | every container's read surface — chiefly Atoms (incl. Projection facet), Orchestration; also Ingestion and Blast Radius report reads |
| `navigating` | Resolving a hierarchical-navigation request | Hierarchical Navigation |
| `analyzing_impact` | Computing or resolving a blast report | Blast Radius |
| `evaluating` | Running a judge / golden set / benchmark / faithfulness verification | Quality |
| `generating` | Calling a model through the gateway | LLM Gateway |
| `authorizing` | Checking the calling session's grant at call admission ([02 — Conformance § Access control](./02-conformance.md#access-control)) | the perimeter, on every admitted call |

## Operations (Tier 1)

Semantics: [10 — Error Model § The standardized Tier-1 vocabulary](./10-error-model.md#the-standardized-tier-1-vocabulary). The vocabulary does double duty as the access-control operation types ([02 — Conformance § Access control](./02-conformance.md#access-control)). The set is mirrored as the `OperationKind` type in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md#errors--the-three-tier-model).

| `operation` | The job |
|---|---|
| `ingest` | Bring new source content into the knowledge layer |
| `query` | Answer a read request over atoms / prose |
| `review` | Work a diff, conflict, blast report, or recorded quality verdict toward resolution |
| `manage_snapshot` | Fork / switch / publish / discard a working snapshot |
| `manage_tree` | Register / rename a tree |
| `maintain` | A system operation — rebuild, integrity sweep, GC |

## Activity end-states

Semantics — the closed-per-activity rule, classification fixed per `(activity, end_state)` pair, the blanket pairs, severity derivation: [10 — Error Model § Activity end-states](./10-error-model.md#activity-end-states). Each row's `severity` derives from its (`fix_domain`, `recoverability`) per [10 § Severity](./10-error-model.md#severity--derived-never-declared).

### Universal end-states

Included in every activity's end-state set ([10 § Universal end-states](./10-error-model.md#universal-end-states)); never repeated in per-method **Errors** declarations.

| `end_state` | The activity ended because… | `fix_domain` | `recoverability` |
|---|---|---|---|
| `contended` | a serialization conflict arose on a concurrent write to the snapshot | `infra` | `retryable` |
| `unavailable` | a dependency it needs is transiently unreachable | `infra` | `retryable` |
| `not_wired` | the capability is optional and not present in this deployment | `client` | `not_recoverable` |
| `unsupported_version` | the caller selected an interface version this implementation does not declare ([11 — Versioning](./11-versioning.md)) | `client` | `user_action` |
| `fault` | an internal invariant was violated mid-activity | `aala` | `not_recoverable` |

### Per-activity end-states

| `activity` | `end_state` | The activity ended because… | `fix_domain` | `recoverability` |
|---|---|---|---|---|
| `ingesting` | `malformed_fragment` | the fragment failed normalization or required-metadata validation | `client` | `user_action` |
| `ingesting` | `unsupported_source` | no normalizer is registered for the fragment's source type | `client` | `user_action` |
| `ingesting` | `model_failure` | the model call behind the relevance filter failed after the gateway's retries — the fragment is neither persisted nor rejected | `infra` | `retryable` |
| `extracting` | `unextractable_content` | the fragment's content yields no parseable claims (a valid extraction of zero atoms is a success, not this) | `client` | `user_action` |
| `extracting` | `schema_violation` | a proposed atom fails the data model's structural contract — the structural side of the schema-violation fence, drawn in [06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes) | `client` | `user_action` |
| `extracting` | `model_failure` | the model call behind extraction failed after the gateway's retries | `infra` | `retryable` |
| `classifying_conflicts` | `model_failure` | an LLM-assisted pipeline stage failed after the gateway's retries | `infra` | `retryable` |
| `transitioning` | `not_found` | the named atom does not exist in the snapshot | `client` | `user_action` |
| `transitioning` | `invalid_transition` | the target outcome is not legal from the atom's current state | `client` | `user_action` |
| `transitioning` | `missing_meta` | a required outcome meta field is absent (e.g. `superseded_by`) | `client` | `user_action` |
| `revising` | `not_found` | the named atom does not exist in the snapshot | `client` | `user_action` |
| `revising` | `schema_violation` | the revised atom fails the structural contract or its own resolved classification/predicate schema — the revision clause of the schema-violation fence ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes)) | `client` | `user_action` |
| `revising` | `immutable_field` | the revision targets a field the contract fixes — `id`, `type`, `status` (mutated only via `update_status`), or `derivation` | `client` | `user_action` |
| `revising` | `not_revisable` | the atom cannot be revised: it is derived (maintained by the entailment engine alone), it is a tombstone in a removal-outcome state, or it is a PredicateKindAtom (its `kind` and `semantics` are fixed by [§ Predicate kinds](#predicate-kinds)) | `client` | `user_action` |
| `snapshotting` | `not_found` | the named snapshot does not exist | `client` | `user_action` |
| `snapshotting` | `discarded` | the target snapshot has been discarded | `client` | `user_action` |
| `snapshotting` | `non_fast_forward` | the canonical pointer moved since the published snapshot's fork — the publish is not a fast-forward; re-fork from the new canonical and replay ([04 § Canonicalization](./04-snapshots.md#canonicalization)) | `client` | `user_action` |
| `snapshotting` | `canonical_protected` | the operation would discard or mutate the canonical snapshot | `client` | `user_action` |
| `managing` | `already_registered` | the tree id is already registered under a differing registration — an identical re-registration is a replay, not an error ([`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency)) | `client` | `user_action` |
| `managing` | `invalid_tree` | the tree id is malformed | `client` | `user_action` |
| `managing` | `not_found` | the named tree is not registered | `client` | `user_action` |
| `querying` | `not_found` | the addressed entity does not exist in the snapshot | `client` | `user_action` |
| `querying` | `invalid_query` | the request is malformed — bad scope path, unknown tree, invalid filter, an over-cap batch, or a paging cursor invalid for the enumeration ([`interfaces/00-shared-types.md § Paged collection reads`](../interfaces/00-shared-types.md#paged-collection-reads)) | `client` | `user_action` |
| `querying` | `stale_ref` | a Delta-Stream `ref` does not name a checkpoint of the calling session's bound snapshot — foreign, discarded, or unrecognized ([05](./05-delta-streams.md)) | `client` | `user_action` |
| `navigating` | `unknown_axis` | the named axis is not registered | `client` | `user_action` |
| `navigating` | `invalid_query` | the request is malformed — a paging cursor invalid for the enumeration ([`interfaces/00-shared-types.md § Paged collection reads`](../interfaces/00-shared-types.md#paged-collection-reads)) | `client` | `user_action` |
| `analyzing_impact` | `not_found` | the origin atom or blast report does not exist | `client` | `user_action` |
| `analyzing_impact` | `invalid_input` | the analysis or resolution request is malformed — an invalid hypothetical state, depth limit, or scope filter; or an unrecognized resolution value or meta field | `client` | `user_action` |
| `analyzing_impact` | `report_closed` | a resolution was recorded against a closed report | `client` | `user_action` |
| `analyzing_impact` | `not_affected` | the atom is not in the report's impact set | `client` | `user_action` |
| `evaluating` | `not_found` | the golden set, call, benchmark version, or a verification-subject atom does not exist | `client` | `user_action` |
| `evaluating` | `invalid_input` | a supplied golden case, feedback payload, evaluation or shadow configuration, verification subject, or paging cursor fails structural validation | `client` | `user_action` |
| `evaluating` | `model_failure` | a judge call failed after the gateway's retries | `infra` | `retryable` |
| `generating` | `unknown_use_case` | the `use_case_key` is not registered | `client` | `user_action` |
| `generating` | `malformed_output` | the model's output failed schema validation after the gateway's internal retries | `infra` | `retryable` |
| `generating` | `throttled` | the provider rate-limited the call (`retry_after_ms` SHOULD be set) | `infra` | `retryable` |
| `generating` | `timeout` | the provider exceeded the call deadline | `infra` | `retryable` |
| `generating` | `provider_failure` | the provider returned an error; persistent recurrence is an operator concern surfaced via telemetry | `infra` | `retryable` |
| `authorizing` | `unauthenticated` | no session is established for the call — the establishment context is absent or invalid ([02 — Conformance § Access control](./02-conformance.md#access-control)) | `client` | `user_action` |
| `authorizing` | `denied` | the session holds no grant for the call's (subject snapshot, operation type) pair ([02 — Conformance § Access control](./02-conformance.md#access-control)) | `client` | `user_action` |
