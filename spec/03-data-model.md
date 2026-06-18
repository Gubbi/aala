# 03 — Data Model

This chapter defines the binding atom data model. The typed shape lives in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md); the spec overview is in [`README.md`](./README.md) and [`00 — Introduction`](./00-introduction.md). This chapter is the normative specification.

## Atom types

An **atom** is the unit of content in a snapshot. Every atom has exactly one of five concrete types:

| Type | Role |
|---|---|
| `entity` | An individual subject the snapshot makes claims about. |
| `relation` | A single triple connecting two atoms. |
| `classification` | A class of subjects plus the structural schema defining what it means to be classified under it. |
| `predicate` | A named predicate. Specialized classification whose instances are RelationAtoms. |
| `predicate_kind` | A kind of predicate carrying intrinsic cascade semantics and hard property characteristics. |

The set of concrete atom types is **closed**. Implementations MUST NOT introduce additional atom types. Extension happens through ClassificationAtoms (adding new classifications) and PredicateAtoms (adding new predicates), not through new atom types.

**What classifies what is fixed by type.** A ClassificationAtom classifies EntityAtoms; a PredicateAtom classifies RelationAtoms. The id-prefix rules on `is_a` enforce this structurally: `entity.is_a` MUST point to a classification, `relation.is_a` MUST point to a predicate.

## Identifier format

Atom and supporting identifiers MUST follow the normative formats specified in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md); minting authority, uniqueness scope, and stability are fixed per identifier type in [Appendix A § Identifier minting](./appendix-a-registries.md#identifier-minting). For atoms specifically:

- `AtomId` has the form `atom:<type>:<opaque>` where `<type>` is one of the five concrete atom type words (`entity`, `relation`, `classification`, `predicate`, `predicate_kind`) and `<opaque>` matches `[A-Za-z0-9_\-.]+`. The binding rule is **total id length ≤ 255**; since the `atom:<type>:` prefix is 12–20 characters (longest for `classification` / `predicate_kind`), `<opaque>` is at least 1 and at most `255 − len("atom:<type>:")` (235 in the worst case).
- The id's `<type>` prefix MUST equal the atom's `type` field. Validators MUST reject atoms where the two disagree.
- The `<opaque>` portion is opaque to the contract — implementations choose UUID, content hash, sequence number, deployment-meaningful slug, etc.

Type-checks on referenced atoms (e.g., verifying an `is_a` target is a ClassificationAtom) reduce to a string-prefix comparison.

## Base Atom fields

Every atom MUST have:

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | `AtomId` | yes | Format per Identifier format above. Globally unique within a snapshot. Stable across snapshots — the same logical atom MUST keep the same `id` through any number of `Updated` events. |
| `type` | `AtomType` | yes | One of the five concrete types. MUST match the id's type prefix. |
| `subject` | string | yes | The atom's identity. Treatment varies by type — see *Subject treatment* below. |
| `is_a` | `AtomId` | conditional | Primary classification (`entity`, `relation`) or superclass (`classification`, `predicate`), single-valued. REQUIRED on `entity`, `relation`, `predicate`, and on every `classification` except the single entity-domain root. Not used on `predicate_kind` (the ten kinds are flat roots; membership is carried by `type`). |
| `attributes` | map | yes | MAY be empty. Holds the **values** for the typed attribute slots the atom's classification declares (`schema.attributes`). Validated for presence/type against the resolved schema. Never used to derive logical consequences. |
| `annotations` | map | yes | MAY be empty. Free-form record metadata. Never schema-validated. The Conflict pipeline, Blast Radius, and entailment engine MUST NOT read `annotations` as input to classification or inference (analogous to OWL `AnnotationProperty`). The reserved key `aala.conflict` is *written* by the Conflict pipeline to record pending situations ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)) — a record, never a reasoning input. |
| `sources` | list of `Source` | yes | Provenance back to originating fragments. MUST be non-empty for atoms extracted from fragments. Accumulates through revision (see [Mutability and revision](#mutability-and-revision)); an atom entering as a refinement or successor of another MAY inherit the prior atom's `Source`s and append its own. |
| `confidence` | number | yes | Range 0.0–1.0. Conflict pipeline assigns. Non-logical metadata — does not gate reasoning. |
| `scope` | `Scope` | yes | Includes `tree`, optional `environment`, optional `time_scope`, and `constraint_atoms` list. |
| `status` | `Status` | yes | Struct `{ state, correlation_id, reason?, changed_at?, meta? }`. See [Status](#status) below and [07 — Atom Lifecycle](./07-atom-lifecycle.md). |
| `derivation` | `Derivation` | no | Present iff the atom was derived (not asserted). Carries `from: AtomId[]` and `rule: string`. See [the entailment section below](#entailment). |

### Additional fields — two extension paths

Two distinct kinds of extension exist; do not conflate them:

- **Implementation / transport fields.** Implementations MAY add additional top-level fields to atoms. Such fields are NOT validated by any classification schema, MUST be carried through `Added` / `Updated` events and preserved on round-trip (per [13 — Wire Profiles](./13-wire-profiles.md) §unknown-field preservation), and MUST NOT change the meaning of the listed fields.
- **Content extension.** Validated, content-level extensibility goes through `schema.attributes` (typed slots a classification declares) and stored as values in the atom's `attributes` map — not through arbitrary top-level fields.

## Per-type field requirements

### `entity`

| Field | Constraint |
|---|---|
| `id` prefix | `atom:entity:` |
| `is_a` | REQUIRED. MUST point to a `ClassificationAtom` (id prefix `atom:classification:`). The entity is an instance of that class. |
| `subject` | Treatment determined by `is_a` target's `schema.subject_kind`. If `named`, MUST be prose. If `referential`, MUST be a valid `AtomId` (and MUST satisfy the schema's `domain_classifications` when set; dead references handled per [12 — Edge Cases](./12-edge-cases.md)). |

### `relation`

| Field | Constraint |
|---|---|
| `id` prefix | `atom:relation:` |
| `is_a` | REQUIRED. MUST point to a `PredicateAtom` (id prefix `atom:predicate:`). |
| `subject` | REQUIRED. MUST be a valid `AtomId` (the from-atom of the edge). MUST satisfy the predicate's `domain_classifications`. |
| `object` | REQUIRED (added field beyond the base shape). MUST be a valid `AtomId` (the to-atom). MUST satisfy the predicate's `range_classifications`. |

The `predicate` semantics (kind, characteristics, domain/range, aliases) are carried by the `PredicateAtom` referenced via `is_a`. RelationAtom carries no separate `predicate` field.

### `classification`

| Field | Constraint |
|---|---|
| `id` prefix | `atom:classification:` |
| `is_a` | Subclass-of, within the same type-domain. MUST point to another `ClassificationAtom`. REQUIRED for every classification **except** the single entity-domain root `Classification("Thing")`, which omits it. The chain MUST terminate at that root. |
| `subject` | MUST be prose — the class name. |
| `schema` | REQUIRED (added field). Carries `subject_kind`, `domain_classifications`, `attributes`, `required_relations`, `optional_relations`, `max_children`, `aliases`, `see_also`, `canonical_for`, `disjoint_with`, optional `definition`. See [Classification schema](#classification-schema) below. |

`is_a` here is subclass-of (subsumption), distinct from the instance-of relation an `entity` has to its class. A classification's membership in the *type* `classification` is carried by its `type` field.

### `predicate`

| Field | Constraint |
|---|---|
| `id` prefix | `atom:predicate:` |
| `is_a` | REQUIRED. MUST point to either a `PredicateKindAtom` (id prefix `atom:predicate_kind:`) OR another `PredicateAtom` (sub-predicate hierarchy). When the chain bottoms out, it MUST terminate at a `PredicateKindAtom`. |
| `subject` | MUST be prose — the predicate name. |
| `schema` | REQUIRED (added field). Carries the classification-shaped fields (`subject_kind` = `"named"`, `aliases`, `see_also`, etc.) PLUS predicate-specific fields: `domain_classifications`, `range_classifications`, `characteristics`, optional `inverse_of`, optional `chain` ([Chain derivation](#chain-derivation)). |

### `predicate_kind`

| Field | Constraint |
|---|---|
| `id` prefix | `atom:predicate_kind:` |
| `is_a` | Not used. The ten kinds are flat roots; membership in the type is carried by `type`. |
| `subject` | MUST be prose — the kind name (one of the ten closed values). |
| `kind` | REQUIRED (added field). One of the ten closed kind values ([Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds)). |
| `semantics` | REQUIRED (added field). Carries `hard_characteristics`, `cascade` rule, and informational `conflict_outcomes` list. See [Predicate kinds](#predicate-kinds) below. |

The set of `kind` values is **closed**. Implementations MUST NOT introduce additional kinds.

The rules of this chapter are enforced on two channels: a defect in the atom's own shape is rejected as a schema-violation Activity Error, while an incompatibility with referenced definitions or coexisting atoms surfaces as a conflict outcome. Which check sits on which side is drawn once — the schema-violation fence, [06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes).

## Subject treatment

The `subject` field carries different content per type:

| Atom type | `subject` is |
|---|---|
| `entity` (when classified under a `named` classification) | Prose — the entity's name (e.g., `"Microsoft"`, `"Alice"`) |
| `entity` (when classified under a `referential` classification) | `AtomId` — pointing at what the claim is about (e.g., a Decision's subject points at the atom the decision is about); constrained by the schema's `domain_classifications` when set |
| `relation` | `AtomId` — the from-atom of the edge |
| `classification` | Prose — the class name (e.g., `"Company"`, `"Technical-Decision"`) |
| `predicate` | Prose — the predicate name (e.g., `"composes"`, `"depends-on"`) |
| `predicate_kind` | Prose — the kind name (e.g., `"composition"`) |

Validation: the Conflict pipeline MUST verify subject treatment matches the type rules above at proposal time. Violations surface as hard-block `InvalidSubject` conflict outcomes — a referenced definition's rule, the logical side of the schema-violation fence ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes)) — results of `propose`, not Activity Errors ([10 — Error Model](./10-error-model.md#conflict-pipeline-outcomes-are-results-not-errors)).

## Classification schema

A `ClassificationAtom.schema` declares the structural contract for atoms classified under it.

| Schema field | Meaning |
|---|---|
| `subject_kind` | `"named"` (classified atoms have prose subjects) or `"referential"` (classified atoms have AtomId subjects). OPTIONAL — absent means **abstract** (a root or intermediate class spanning both). Once a chain declares it, every descendant MUST match; it cannot change below the declaration point. |
| `domain_classifications` | Referential classifications only: the allowed classes of the subject reference (the referent). Absent/empty = open (any referent). N/A for `named`. Same field — generalized — as a predicate's `domain_classifications`, which constrains a RelationAtom's from-atom. |
| `attributes` | Typed attribute slots. Each declares `key`, `value_type`, `required`. Instances carry the values in their `attributes` map. Inherited cumulatively from parents. |
| `required_relations` | Existential + cardinality restrictions: each declares `predicate` (PredicateAtom), `target_classification`, `min`, `max`. Classified atoms MUST satisfy. Inherited cumulatively. |
| `optional_relations` | Same shape with `min` = 0. |
| `max_children` | Direct-children count limit (default 20; deployment-overridable). Sub-categories + classified instances combined. Forces broad concept sets to introduce grouping levels rather than sprawling flat under a single parent. |
| `aliases` | Surface-form variations of the class name. Used by extraction for matching. |
| `see_also` | Related classifications. Presentation only; never used for resolution. |
| `canonical_for` | Optional reverse pointer marking this as the canonical name when multiple overlap. |
| `disjoint_with` | List of incompatible classifications. Applies within-tree only (see [06](./06-conflict-classification.md)). |
| `definition` | Optional prose definition for glossary projection. |

Inheritance rules:

- Children may **add** attributes, `required_relations`, `optional_relations`, and `disjoint_with` entries — never remove parent's.
- `subject_kind` is invariant **below the point it is first declared**: a child MUST NOT contradict an ancestor that fixed it; an abstract ancestor (no `subject_kind`) lets a descendant fix it once for that sub-tree.
- Children's `max_children` may override the parent's (raise or lower).

Validation walks the `is_a` chain to the root and accumulates schema fields. Atoms classified under a chain MUST satisfy the cumulative schema. Violations of the cumulative schema surface as conflict outcomes (pipeline stage 3 — [06 § Outcome registry](./06-conflict-classification.md#outcome-registry)), on the logical side of the schema-violation fence ([06 § Malformed input vs. conflict outcomes](./06-conflict-classification.md#malformed-input-vs-conflict-outcomes)); on the direct revision surface the atom's own resolved schema is enforced by the revision gate (`revising` / `schema_violation` — [`interfaces/atoms.md § update`](../interfaces/atoms.md#update)).

## Predicate kinds

The closed set of `kind` values is **ten**, organized as inverse pairs (two self-inverse). Each kind carries hard characteristics, a cascade rule, and an `inverse_of` pointer to its inverse kind; the registry — every kind with its edge meaning, hard characteristics, cascade trigger, and inverse — is [Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds). Edges are written subject → object.

The `reflexive` characteristic is intentionally not modeled.

The registry's **Cascade** column summarizes the per-kind triggers informatively. The normative definition of impact propagation — trigger application, channel ordering, fixpoint, confidence gating, and emissions — is [07 — Atom Lifecycle § Cascade rules](./07-atom-lifecycle.md#cascade-rules).

**Inverse pairing is a kind-level property.** A predicate's inverse is itself a predicate of the inverse kind: if PredicateAtom `P is_a K` declares `inverse_of = Q`, then `Q is_a K.inverse_of` (T1 materializes the reverse edges). The `is_a` *field*, multi-classification edges, and claim-refinement all run in the `specialization` direction (subject = the more specific atom). The characteristic and cascade entries in the registry are the inversion of each pair: `functional` ↔ `inverse_functional` swap, and the cascade direction reverses (source↔target).

Hard characteristics MUST hold for every PredicateAtom whose `is_a` chain bottoms out at the given kind. Soft characteristics (`functional`, `inverse_functional` where not hard) MAY be set on the PredicateAtom by the extractor; the Conflict pipeline validates them.

**The composition family is not transitive.** `composition` and `component_of` carry exclusivity constraints (`inverse_functional` / `functional`); a transitive kind carrying exclusivity constraints would manufacture violations out of its own closure — every legal two-level hierarchy (A composed-of B, B composed-of C) would entail a second incoming edge on C. The two kinds therefore describe **direct** parthood only, and no derived RelationAtom ever carries a composition-family kind: T2 computes no closure for them, and a chain-declaring predicate **MUST NOT** bottom out at `composition` or `component_of` ([Chain derivation](#chain-derivation)). Transitive containment is expressed through explicitly declared chain predicates of the aggregation family; the standard library ships `part_of` / `has_part` for exactly this ([Standard library](#standard-library--pre-registered-atoms-per-tree)).

**Characteristics are validated over asserted edges only.** The cardinality-like characteristics (`functional`, `inverse_functional`) count **asserted** RelationAtoms; derived RelationAtoms never count toward them. This is a general backstop, not a composition-family special case: it holds for any transitive kind on which a deployment sets a soft cardinality-like characteristic. Violations of `asymmetric` / `irreflexive` are likewise violations of asserted structure — the entailment closure **MAY** be used to *detect* a cycle, but the atoms a violation cites are the asserted edges forming it ([06 — Conflict Classification](./06-conflict-classification.md)).

Composition's `inverse_functional` is hard: an atom **MUST NOT** be the target of more than one active **asserted** composition edge (the single-parent rule). A second incoming asserted composition edge triggers `CompositionConflict`. Its inverse, `component_of`, carries the same constraint as a hard `functional` (each part has at most one whole).

## Status

`status` is a `Status` struct: `{ state, correlation_id, reason?, changed_at?, meta? }`. Detailed transition rules are in [07 — Atom Lifecycle](./07-atom-lifecycle.md).

| Struct field | Notes |
|---|---|
| `state` | One of the present states or removal outcomes below. |
| `correlation_id` | The operation responsible for the current state (`ingest:` / `request:` / `system:`). Equals the `correlation_id` of the change event that set it. Non-logical audit metadata — materialized on the atom so "which operation drove this state" is answerable without scanning the Delta Stream; MUST NOT drive reasoning. |
| `reason` | Optional human-readable rationale for the current state. |
| `changed_at` | RFC 3339 timestamp ([shared-types § Timestamps and durations](../interfaces/00-shared-types.md#timestamps-and-durations)). REQUIRED when `state` is not the initial `active`. |
| `meta` | Removal-outcome metadata (see below) and, when a conflict-pipeline resolution drove the state, the outcome classification — `outcome_kind` / `outcome_stage` / `outcome_confidence` ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)); non-logical audit metadata. |

Present states:

| State | Meaning |
|---|---|
| `active` | Currently held. The default for new atoms. |
| `hanging` | Premise (via `constraint_atoms` or via a cascading source) may have changed; awaiting resolution. |
| `deferred` | Known to be hanging; team chose to defer. |
| `deprecated` | Still holds for existing dependents; no new dependencies should be added. |

Removal outcomes (the atom transitions to one of these and is retained as a tombstone; the transition is recorded as an event in the Atoms Delta Stream — see [05 — Delta Streams](./05-delta-streams.md)):

| Outcome | `meta` required |
|---|---|
| `superseded` | `meta.superseded_by` REQUIRED. |
| `duplicate` | `meta.duplicate_of` REQUIRED. |
| `rejected` | — |
| `removed` | — |

`superseded` is the **succession** outcome and is gated: it records that a different claim fully absorbed this one as a compatible refinement, and asserted references to the superseded atom resolve to its successor. The gate, the resolution invariant, and the choice among the removal outcomes when a claim is displaced are defined in [07 — Atom Lifecycle § Revision, succession, and the prior atom's status](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status).

Removing an atom that **was committed** is a **two-step process**. First, the atom transitions to the removal-outcome `state` and that transition is **committed** — the atom is **not** deleted; it remains as a **tombstone** carrying the removal-outcome state plus its `status.correlation_id`, `reason`, and `meta`. Physical deletion is never the direct result of a removal outcome. An **optional garbage-collection (GC) step** MAY later purge tombstones per a deployment retention policy (see [07 — Atom Lifecycle](./07-atom-lifecycle.md) § Garbage collection). This keeps a clean, auditable lifecycle for every committed atom: its removal is a recorded state change, not a silent disappearance.

An atom that was **never committed** is not tombstoned — a proposal `rejected` by the pipeline, or a proposal-time `duplicate` / conflict caught before the atom entered, is simply not added (no tombstone), and such outcomes MAY auto-resolve. The non-entry is recorded as a `Rejected` audit event regardless ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)).

## Provenance — `Source`

```typescript
interface Source {
  uri:            string  // source artifact URI
  quote:          string  // supporting text from the source
  extracted_at:   string  // RFC 3339
}
```

Every atom extracted from a fragment MUST have at least one `Source`. Provenance accumulates through in-place revision per [Mutability and revision](#mutability-and-revision). An atom entering as a refinement or successor of another MAY inherit the prior atom's `Source`s and append the ones that justify it. Derived atoms inherit `Source`s from their `derivation.from` atoms.

## Mutability and revision

Atoms are **mutable**. An in-place content update is the primary revision path: the atom's `id` is stable across edits, every revision is recorded as an `Updated` event carrying the full before/after states ([`interfaces/atoms.md`](../interfaces/atoms.md#update)), and the working-snapshot diff shows an edit as an edit of one atom ([04 — Snapshots](./04-snapshots.md#atom-identity-across-snapshots)). RelationAtoms are equally mutable — endpoint and attribute corrections revise the edge in place. The model carries no history machinery: previously published snapshots carry what was, and historical reconstruction is out of scope ([00 — Introduction](./00-introduction.md#whats-out-of-scope)).

Three field groups are excluded from revision:

| Excluded | Why |
|---|---|
| `id`, `type` | Identity. Immutable for the atom's lifetime. |
| `status` | Owned by the lifecycle. Mutated only through `Atoms.update_status` ([07 — Atom Lifecycle](./07-atom-lifecycle.md)); a revision never changes status. |
| `derivation` — and every field of a derived atom | Derived atoms are maintained by the entailment engine alone; they are never revised directly. PredicateKindAtoms are likewise never revised — their `kind` and `semantics` are fixed by the predicate-kind registry ([Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds)). |

### The provenance-fidelity rule

Whether a change is a *revision of this atom* or a *different claim* is decided by one test:

> An atom **MAY** be revised in place **iff** the revised content remains an honest record of the atom's own provenance — including corrected readings of that provenance.

Typo and wording fixes, extraction corrections (the fragment was misread), and referent-description updates all pass: the same sources, read correctly, support the revised content. A change that needs new, **contrary** provenance to back it fails the test — it is a different claim and **MUST** enter as a new atom via `Atoms.propose`, where the Conflict pipeline classifies it against the existing one ([06 — Conflict Classification § Revision and succession](./06-conflict-classification.md#revision-and-succession)); the prior atom's status is then resolved per [07 § Revision, succession, and the prior atom's status](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status).

Provenance accumulates through revision: a revision justified by a new fragment **MUST** append that fragment's `Source`, and a revision **MUST NOT** drop `Source`s the revised content still rests on. A pure reading correction needs no new `Source`.

### Per-type revision defaults

| Atom kind | Revision default |
|---|---|
| Identity anchors — EntityAtoms, ClassificationAtoms, definition-bearing atoms (glossary terms) | Revise in place. Succession applies only to identity repair — a conflation merge or split — and is reviewer-driven. |
| RelationAtoms | Revise in place for corrections (a misextracted endpoint, a wrong attribute value). When the *world* rewired — the old edge genuinely held and no longer does — the new edge enters as a new atom and the old one transitions per [07](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status). |
| Premise-bearing atoms — ConstraintAtoms and any atom referenced from `Scope.constraint_atoms` | Wording fixes only in place. A semantic change is a new atom plus the old one `deprecated` / `removed`, so the scope-premise cascade channel re-validates everything gated on it ([07 § Channel 2](./07-atom-lifecycle.md#channel-2-scope-premise-cascade)). |

### Schema-carrying atoms

A revision that changes a ClassificationAtom's or PredicateAtom's `schema` is a **governed mutation**: beyond ordinary structural validation it MUST be validated against the schema-inheritance rules ([Classification schema](#classification-schema): children only add, `subject_kind` invariance below its declaration point) and the chain/kind restrictions ([Chain derivation](#chain-derivation)), and it emits the type-specific schema event in addition to `Updated` ([`interfaces/atoms.md`](../interfaces/atoms.md#changes_since)). Consequences for atoms already classified under the revised schema surface through the Conflict pipeline's entailment-recompute trigger ([06](./06-conflict-classification.md#when-the-pipeline-runs)).

### Revisions never cascade

A content revision is not a status transition and hangs nothing — the cascade is status-driven ([07 § Cascade rules](./07-atom-lifecycle.md#cascade-rules)). The revision result lists dependents as **advisory**; assessing the semantic impact of changed content is [Blast Radius](./08-blast-radius.md)'s on-demand job. A revision to logically significant fields (`is_a`, a RelationAtom's `subject` / `object`, `scope`, `schema`) re-runs entailment for affected derivations within the same operation, so derived atoms stay consistent with their sources ([04 § Derived atoms within a snapshot](./04-snapshots.md#derived-atoms-within-a-snapshot)).

The revision surfaces — the direct `Atoms.update` method and the pipeline-applied path — are specified in [`interfaces/atoms.md`](../interfaces/atoms.md#update) and [06 § Revision and succession](./06-conflict-classification.md#revision-and-succession).

## Scope

```typescript
interface Scope {
  tree:              TreeId           // bounded discourse
  environment?:      string
  time_scope?:       TimeScope
  constraint_atoms:  AtomId[]         // ConstraintAtoms (entities classified under "Constraint") gating this atom
}
```

| Field | Logical or non-logical | Notes |
|---|---|---|
| `tree` | Logical | The autonomous existence this atom belongs to (see below). Partitions same-claim Duplicate detection. Cross-tree similarity surfaces as candidate `kind=equivalence` link. |
| `environment` | Non-logical | Metadata. |
| `time_scope` | Non-logical | An atom with `time_scope.effective_to < now()` MAY be filtered from active reads. The atom remains in the snapshot. |
| `constraint_atoms` | Logical | Each entry is an AtomId pointing to a ConstraintAtom. When any listed constraint transitions to a non-`active` state, this atom transitions to `hanging` via the cascade. Equivalent to a dependency RelationAtom from this atom to each listed constraint, but stored efficiently on Scope. |

A **tree** is an autonomous existence: a bounded discourse with its own stakeholders, owners, and lifecycle — e.g., a design/ADR tree evolving independently of the deployed-implementation tree describing the same system. Tree partitioning bounds same-claim Duplicate detection and disjointness ([06](./06-conflict-classification.md)); it does not silo claims — asserted RelationAtoms MAY connect atoms across trees, and such edges cascade normally ([07 § Tree boundaries and cascade](./07-atom-lifecycle.md#tree-boundaries-and-cascade)). `kind=equivalence` links map similar concepts across autonomous trees; they record correspondence, never transmit automatic transitions.

## Non-logical channels

The following channels carry metadata only — the Conflict pipeline, Blast Radius, and entailment engine MUST NOT use them to derive logical consequences:

| Channel | Role |
|---|---|
| `annotations` | Free-form record metadata; never validated (annotation-property analog). The reserved `aala.conflict` key carries pending conflict records ([06 § Outcome recording](./06-conflict-classification.md#outcome-recording)) — written and cleared as a record, never read for reasoning. |
| `attributes` | Typed attribute **values**; validated for presence/type against the classification schema, but never used to derive logical consequences. |
| `confidence` | Numeric belief. Used by Conflict pipeline as a threshold input (e.g., composition cascade gating) but not as a logical proposition. |
| `Scope.time_scope` | Temporal metadata. |
| `Scope.environment` | Deployment metadata. |

## Entailment

Implementations MUST compute three tiers of entailment. Implementations MAY choose caching strategy (lazy, eager, or hybrid).

| Tier | Rules |
|---|---|
| **T1** | Transitive closure of `is_a` (an entity's instance-of plus its class's subclass-of chain); `inverse_of` materialization; aliases transitive closure. |
| **T2** | Transitive closure of RelationAtoms whose PredicateKind carries the hard `transitive` characteristic ([Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds) — never `association` or the composition family); chain derivation over declared chain predicates ([Chain derivation](#chain-derivation)); symmetric expansion for `equivalence`; cross-tree equivalence implications. |
| **T3** | Disjointness propagation through `is_a` chains; `functional` / `inverse_functional` violations through chains. T3 produces Conflict outcomes only — no new atoms. |

T1 + T2 produce derived atoms; each carries a populated `derivation` field. Derived atoms are hidden by default in the Projection facet; opt-in surfacing only.

When an asserted atom and a derived atom collide on the same `(subject, predicate-via-is_a, object, scope)`, the asserted atom wins; the derived one is dropped.

When a source atom (an atom listed in some derived atom's `derivation.from`) transitions to a non-`active` state, every derived atom referencing it in `derivation.from` MUST be invalidated and either recomputed or removed.

## Chain derivation

Transitive reach over non-transitive structure is **declared, never assumed**. A PredicateAtom **MAY** carry a `chain` declaration in its schema — an ordered sequence of steps, each naming a concrete PredicateAtom with an optional one-or-more repetition marker (`+`). The standard-library declaration reads:

```
part_of := component_of ∘ component_of+
```

— a `part_of` edge is derived for every path of two or more `component_of` edges. The rules:

- **Declaration shape.** `chain` is an ordered list of steps `{ predicate, repeat? }` (typed shape in [`00-shared-types.md`](../interfaces/00-shared-types.md)). A step **MUST** name a **PredicateAtom** — never a PredicateKindAtom. Chains bind concrete predicates, not kinds: a deployment predicate `chapter_of` (kind `component_of`) participates only in chains that name it or an ancestor it sub-predicates — kind-mates never smear into each other's chains.
- **Derivation.** A RelationAtom `(a → b)` of the declaring predicate is derived whenever a path of edges from `a` to `b` matches the step sequence: each step is matched by one edge — or, for a `+` step, by one or more consecutive edges — whose predicate (via `is_a`) is the step's predicate or a sub-predicate of it.
- **Matching domain.** Chain matching runs over **asserted** RelationAtoms and their T1 inverse-materialized counterparts (one assertion read in either direction). T2-derived edges — transitive closures and chain-derived edges — never participate in matching; derivation is a single, non-recursive pass over asserted structure.
- **Derived-edge shape.** The derived RelationAtom carries the **declaring** predicate as its `is_a` (and therefore the declaring predicate's kind); its `subject` is the path's first atom, its `object` the last. `derivation.from` lists the matched edge RelationAtoms in path order; `derivation.rule` is `"predicate-chain"`. Channel-3 invalidation applies: the derived edge **MUST** be invalidated when any matched edge leaves `active` ([07 — Atom Lifecycle § Channel 3](./07-atom-lifecycle.md#channel-3-derivation-invalidation)).
- **No declaration ⇒ no derivation.** There is no global containment or closure rule beyond T2's per-kind transitivity. Absent a chain declaration, no chain-derived edge exists.
- **Kind restriction.** A chain-declaring PredicateAtom **MUST NOT** have an `is_a` chain bottoming out at `composition` or `component_of`. Together with the composition family's non-transitivity, this makes a derived composition-family edge structurally unrepresentable — the exclusivity constraints (`functional` / `inverse_functional`) can never collide with derived structure. A violating declaration surfaces as a `CharacteristicKindConflict` conflict outcome ([06 — Conflict Classification](./06-conflict-classification.md)).
- **Not inherited.** A chain declaration binds only the declaring PredicateAtom; sub-predicates do not inherit it.
- **Cascade-inert.** Like every derived edge, chain-derived edges never transmit a cascade ([07 — Atom Lifecycle § Cascade rules](./07-atom-lifecycle.md#cascade-rules)). They serve queries and navigation.

**Aggregation-read default.** Reads filtered to aggregation-family predicates or kinds (`aggregation` / `member_of`) include derived containment edges **by default** — the inverse of the general hidden-by-default rule for derived atoms — because derived containment is what such queries exist to answer. Callers exclude derived edges by inspecting `derivation` or via the read surface's explicit derived-atom toggle ([`interfaces/atoms.md` § `traverse_relations`](../interfaces/atoms.md#traverse_relations)).

## Standard library — pre-registered atoms per tree

Every tree MUST initialize with the standard library enumerated in [Appendix A § Standard library](./appendix-a-registries.md#standard-library): the ten PredicateKindAtoms (one per `kind` value, each a flat root with no `is_a`), the standard-library ClassificationAtoms, and the standard-library PredicateAtoms.

Deployment-defined composition-family predicates participate in `part_of` derivation only by opting in: either declared as sub-predicates of the standard `component_of` predicate (chain steps match sub-predicates) or carrying their own chain-declaring predicates. A composition-family predicate registered directly under the kind stays outside every chain it is not named in.

Deployments MAY register additional classifications and predicate atoms but MUST preserve the standard library above.

## Schema extensibility

Versioning is governed by [11 — Versioning](./11-versioning.md). Backward-compatible additions are:

- New ClassificationAtoms.
- New PredicateAtoms.
- Additional attribute slots in ClassificationAtom schemas.
- Additional optional fields on atoms.

Breaking changes include:

- Adding or removing concrete atom types.
- Adding or removing predicate kinds.
- Removing or repurposing existing fields.
- Changing schema inheritance rules.
- Changing the meaning of an existing kind's cascade or hard characteristics.
- Changing the identifier format.

Pre-publish (v0.x), in-place overwrite is permitted at the spec level.
