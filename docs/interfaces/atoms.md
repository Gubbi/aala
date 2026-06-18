# Atoms — Interface Contract

**Version:** `aala-v0.1`
**Optionality:** non-optional (every aala deployment exposes Atoms)

Atoms is the canonical record. It owns the atom schema, the durable store, extraction, conflict detection, status transitions, classification and predicate resolution, entailment computation, and the indices Conflict consults internally.

Atoms also implements the [Projection facet](./projection.md) (required) over its claims — canonical claim prose, the glossary, and a navigable index. That read-only facet is a second read surface alongside the structured methods below; its contract lives in [`projection.md`](./projection.md).

## Methods

### `propose`

```typescript
function propose(
  fragment:         NormalizedFragment,
  extractor_hints?: ExtractorHints
): Promise<ProposeResult>
```

Runs extraction → conflict classification → atomic mutation of the snapshot. Accepted atoms are added; proposals the pipeline classifies as **revisions** of existing canonical atoms are applied as in-place updates of those atoms — `Updated` events, no new atom ([06 § Revision and succession](../spec/06-conflict-classification.md#revision-and-succession)); affected atoms are mutated through Status Manager; cascades across all three channels (predicate-kind, scope-premise, derivation invalidation) are applied to fixpoint. The result is a **summary**: structural id lists plus the pipeline's classification buckets. There is no conflict-outcome entity — pending situations are recorded in atom-space and reviewed through the ordinary read surfaces ([06 § Outcome recording](../spec/06-conflict-classification.md#outcome-recording)).

```typescript
interface ExtractorHints {
  prefer?:          string[]    // extractor ids to prefer
  exclude?:         string[]    // extractor ids to skip
  content_kind?:    ContentKind // override the fragment's content_kind for routing
  target_tree?:     TreeId      // override the destination tree for extracted atoms
}

interface ProposeResult {
  ingest_id:        IngestId
  added:            AtomId[]                       // created: atoms committed as new (asserted)
  updated:          AtomId[]                       // canonical atoms revised in place by revision-class resolutions (spec/06 § Revision and succession); each emitted an Updated event
  derived_added:    AtomId[]                       // advisory — T1/T2 derived atoms the operation MATERIALIZED; lazy caching MAY report none regardless of derivable consequences (spec/12 § Derived atoms at the read and result surfaces)
  conflicts:        ConflictSummary                // the pipeline's classification summary — counts + buckets (spec/06 § Outcome recording)
  cascaded:         AtomCascadeRecord[]            // atoms that transitioned status via cascade
  advisory_dependents: AtomId[]                    // dependents of the atoms in `updated`, via cascade-bearing structure — advisory only; content revisions cascade nothing (spec/07 § Cascade rules)
}

interface ConflictSummary {
  counts: {                                        // every count MUST equal the size of the bucket it summarizes:
    created:        number                         //   == added.length
    updated:        number                         //   == updated.length
    hanging:        number                         //   == |{ r ∈ cascaded : r.to_status = "hanging" }|
    duplicate:      number                         //   == duplicate.length
    superseded:     number                         //   == superseded.length
    auto_resolved:  number                         //   == auto_resolved.length
    flagged:        number                         //   == flagged.length
    hard_blocked:   number                         //   == hard_blocked.length
  }
  duplicate:        ConflictRecord[]               // canonical atoms that absorbed a duplicate proposal (dropped, source-appended, or applied as a revision)
  superseded:       ConflictRecord[]               // prior atoms transitioned to `superseded` through the succession gate
  auto_resolved:    ConflictRecord[]               // situations the pipeline closed itself by deterministic resolution
  flagged:          ConflictRecord[]               // soft-prompt situations recorded in atom-space, pending review (spec/06 § Outcome recording)
  hard_blocked:     BlockedCandidate[]             // candidates rejected and NEVER persisted — returned here in full
}

interface ConflictRecord {
  atom_id:          AtomId                         // the primary affected persisted atom
  outcome_kind:     ConflictOutcomeKind            // classification vocabulary — registry in spec Appendix A § Conflict outcomes; semantics in spec/06
  involved:         AtomId[]                       // the other persisted atoms in the situation
  linkage_id?:      AtomId                         // the conflicts_with RelationAtom recording a pending multi-atom situation, when one was created
  rationale:        string
  resolution_paths: ResolutionPath[]               // recommended paths (flagged bucket; empty elsewhere)
  meta:             Record<string, unknown>        // outcome-specific (e.g., superseded_by, duplicate_of)
}

interface BlockedCandidate {
  candidate:        Atom                           // the rejected candidate as extracted — never committed, no tombstone (spec/07 § Atoms in removal outcomes)
  outcome_kind:     ConflictOutcomeKind
  involved:         AtomId[]                       // existing atoms implicated (e.g., both composition edges)
  rationale:        string
  resolution_paths: ResolutionPath[]
  meta:             Record<string, unknown>
}

type ConflictOutcomeKind =
  | "Duplicate"
  | "Refines"
  | "Generalizes"
  | "Supersedes"
  | "CompositionConflict"
  | "RelationKindConflict"
  | "GeneralizationCycle"
  | "GeneralizationEquivalenceConflict"
  | "GeneralizationAliasConflict"
  | "WrongCategoryTypeConflict"
  | "CategoryMaxChildrenExceeded"
  | "EquivalenceVsAlias"
  | "SameTreeEquivalenceConflict"
  | "DisjointEquivalenceConflict"
  | "DisjointGeneralizationConflict"
  | "DuplicateClassification"
  | "PromoteClassificationLeafToHub"
  | "MissingRequiredAttribute"
  | "AttributeTypeViolation"
  | "MissingRequiredRelation"
  | "RelationCardinalityExceeded"
  | "RelationDomainViolation"
  | "SubjectClassificationViolation"
  | "InvalidSubject"
  | "CrossTreeEquivalenceCandidate"
  | "FunctionalViolation"
  | "InverseFunctionalViolation"
  | "AsymmetryViolation"
  | "InvalidEquivalenceEndpoint"
  | "IrreflexivityViolation"
  | "CharacteristicKindConflict"
  | "SubjectKindInheritanceConflict"
  | "DeadEdge"
  | "DeadConstraint"
  | "AliasCollision"

interface ResolutionPath {
  label:   string                                  // human-readable description
  action:  string                                  // machine-readable action id
  meta:    Record<string, unknown>
}

interface AtomCascadeRecord {
  atom_id:       AtomId
  from_status:   AtomStatus
  to_status:     AtomStatus
  channel:       "predicate-kind" | "scope-premise" | "derivation-invalidation"
  triggered_by:  AtomId                            // upstream atom whose change caused this cascade
  rationale:     string
}
```

**The reserved conflict annotation.** A pending (soft-prompt) situation is recorded in atom-space under the reserved annotation key `aala.conflict` — on the `conflicts_with` linkage RelationAtom for situations relating two or more persisted atoms, or on the affected atom itself for single-atom situations. The value is the `ConflictRecord[]` shape above (an atom or linkage MAY carry several pending records). The recording semantics, the review surfaces, and the resolution paths are normative in [06 § Outcome recording](../spec/06-conflict-classification.md#outcome-recording); implementations MUST NOT use the `aala.conflict` key for anything else.

**Errors:** `extracting` — `unextractable_content` (the fragment yields no parseable claims; a valid extraction of zero atoms is a success, not this), `schema_violation` (a proposed atom fails the data model's structural contract — the structural side of the schema-violation fence, [06 § Malformed input vs. conflict outcomes](../spec/06-conflict-classification.md#malformed-input-vs-conflict-outcomes); the Primitive `cause` names the offending field), `model_failure` (the extraction model call failed after the gateway's retries); `classifying_conflicts` — `model_failure` (an LLM-assisted pipeline stage failed after the gateway's retries).

**Access:** `ingest` ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

**Idempotency:** idempotent on `fragment.message_id` within the bound snapshot (key-identified) — a reused id is a replay: a full no-op returning the recorded `ProposeResult` (or a semantically equivalent one). Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per snapshot.

A hard-blocked candidate is returned in `conflicts.hard_blocked` — in full, because it is NOT added to the snapshot and leaves no tombstone; the only persistent trace is a `Rejected` audit event carrying the candidate ([06 § Outcome recording](../spec/06-conflict-classification.md#outcome-recording)). Soft-prompt situations leave the proposal in the snapshot (asserted), are reported in `conflicts.flagged`, and are recorded in atom-space for later review; auto-resolved situations are closed within the operation and reported in `conflicts.auto_resolved` for record only.

---

### `update_status`

```typescript
function update_status(
  atom_id:    AtomId,
  outcome:    StatusOutcome,
  rationale:  string,
  meta?:      StatusOutcomeMeta
): Promise<UpdateStatusResult>
```

Applies a present-state transition or a removal outcome to an atom. All such mutations go through the Status Manager (the only mutator of canonical atom state). The full cascade fixpoint is run before return.

```typescript
type StatusOutcome =
  | "active"
  | "hanging"
  | "deferred"
  | "deprecated"
  | "superseded"
  | "duplicate"
  | "rejected"
  | "removed"

interface StatusOutcomeMeta {
  superseded_by?:     AtomId
  duplicate_of?:      AtomId
  blast_id?:          BlastId            // when triggered by a Blast Radius resolution
  idempotency_key?:   string             // optional caller-supplied key
}

interface UpdateStatusResult {
  atom_id:         AtomId
  from_status:     AtomStatus
  to_status:       StatusOutcome
  cascaded:        AtomCascadeRecord[]   // atoms that transitioned via cascade as a consequence
}
```

**Errors:** `transitioning` — `not_found` (the named atom does not exist in the snapshot), `invalid_transition` (the target outcome is not legal from the atom's current state), `missing_meta` (a required outcome meta field is absent).

`superseded` and `duplicate` outcomes REQUIRE the corresponding meta field; absence fails with `transitioning` / `missing_meta`. The `superseded_by`, `duplicate_of`, and `blast_id` fields persist into the atom's `status.meta` (`StatusMeta` in [00 — Shared Types § Atom (base shape)](./00-shared-types.md#atom-base-shape)) — the transition's audit record; `idempotency_key` is call plumbing and is not persisted.

**Access:** `review`.

**Idempotency:** idempotent on `(atom_id, outcome, meta.idempotency_key?)` — an equivalent replay, notably a call naming the atom's current state, is a full no-op: state, `status.changed_at`, and the Delta Stream are untouched. A call naming a different outcome is an ordinary transition under [07 — Atom Lifecycle § API for transitions](../spec/07-atom-lifecycle.md#api-for-transitions), never an overwrite. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per atom within a snapshot.

---

### `update`

```typescript
function update(
  atom_id:    AtomId,
  changes:    AtomRevision,
  rationale:  string
): Promise<UpdateResult>

interface AtomRevision {
  subject?:      string                            // prose or AtomId per the atom's subject treatment (spec/03)
  object?:       AtomId                            // RelationAtoms only
  is_a?:         AtomId                            // re-classification; same id-prefix rules as at proposal
  attributes?:   Record<string, unknown>
  annotations?:  Record<string, unknown>
  sources?:      Source[]                          // the full new list; provenance-accumulation rules in spec/03 § Mutability and revision
  confidence?:   number
  scope?:        Scope
  schema?:       ClassificationSchema | PredicateSchema   // classification / predicate atoms only — governed mutation
}

interface UpdateResult {
  atom_id:              AtomId
  after:                Atom                       // the atom as revised
  advisory_dependents:  AtomId[]                   // atoms depending on the revised atom via cascade-bearing structure — advisory only; nothing transitioned
  derived_invalidated:  AtomId[]                   // advisory — MATERIALIZED derived atoms the recompute invalidated (spec/12 § Derived atoms at the read and result surfaces)
  derived_added:        AtomId[]                   // advisory — derived atoms the recompute (re)materialized; lazy caching MAY report none
}
```

Applies an in-place content revision to one atom — the direct mutation surface beside the pipeline-applied path ([06 § Revision and succession](../spec/06-conflict-classification.md#revision-and-succession)). Each field present in `changes` replaces the atom's field in full (replace, not merge); absent fields are untouched. The revised atom is validated structurally and against its resolved classification/predicate schema before any mutation; a revision to logically significant fields (`is_a`, `subject` / `object`, `scope`, `schema`) re-runs entailment for affected derivations within the same operation, and the T3 checks re-run on that recompute, surfacing conflict outcomes per [06](../spec/06-conflict-classification.md#when-the-pipeline-runs).

A revision never changes `status` and never cascades — dependents are returned as `advisory_dependents` ([spec/07 § Cascade rules](../spec/07-atom-lifecycle.md#cascade-rules)). Whether in-place revision is the right surface at all is governed by the provenance-fidelity rule ([spec/03 § Mutability and revision](../spec/03-data-model.md#mutability-and-revision)); content that amounts to a different claim goes through `propose`.

A `schema` revision on a ClassificationAtom / PredicateAtom is a **governed mutation**: it is additionally validated against the schema-inheritance rules and chain/kind restrictions ([spec/03](../spec/03-data-model.md#schema-carrying-atoms)) and emits `ClassificationSchemaUpdated` / `PredicateSchemaUpdated` after the `Updated` event for the same revision.

**Errors:** `revising` — `not_found` (the named atom does not exist in the snapshot), `schema_violation` (the revised atom fails the structural contract or its own resolved classification/predicate schema — the revision clause of the schema-violation fence, [06 § Malformed input vs. conflict outcomes](../spec/06-conflict-classification.md#malformed-input-vs-conflict-outcomes); the Primitive `cause` names the offending field), `immutable_field` (the revision targets a field outside `AtomRevision` that the contract fixes — `id`, `type`, `status`, `derivation`), `not_revisable` (the atom is derived, is a tombstone in a removal-outcome state, or is a PredicateKindAtom).

**Access:** `review`.

**Idempotency:** state-equality — a call whose `changes` leave every targeted field equal to its current value is a full no-op (no mutation, no events). There is no idempotency key; a repeated call that changes values is a new revision. Registry row in [00 — Shared Types § Idempotency](./00-shared-types.md#idempotency).

**Concurrency:** serialized per snapshot (a write operation — [spec/04 § Operation isolation](../spec/04-snapshots.md#operation-isolation-and-concurrency)).

---

### `get_by_id`

```typescript
function get_by_id(atom_id: AtomId): Promise<Atom | null>
```

Returns the atom in the current snapshot, or `null` if not present. Atoms in removal-outcome states are tombstones and ARE returned (with their removal-outcome `status`); `null` is returned only for ids that never existed or whose tombstone has been purged by garbage collection.

Returns asserted and derived atoms alike. Derived atoms have `derivation` populated.

**Errors:** `querying` — universal end-states only (absence is `null`, not an error).

**Access:** `query`.

**Concurrency:** concurrent-safe.

---

### `get_by_ids`

```typescript
function get_by_ids(atom_ids: AtomId[]): Promise<(Atom | null)[]>
```

Batch read: returns one entry per input id, **positionally aligned** — `result[i]` is the atom for `atom_ids[i]`, or `null` under the same absence rules as `get_by_id` (tombstones ARE returned; `null` only for ids that never existed or whose tombstone has been purged). Duplicate input ids are answered per position. The batch is read from one consistent view of the bound snapshot.

This is the hydration path for provenance- and traversal-following consumers (a `ProjectionDoc.provenance` list, a `via` edge chain, a page of `node_atoms`): one call per id list, not one call per id.

Implementations MUST accept at least 1 000 ids per call; a batch exceeding the implementation's cap fails with `querying` / `invalid_query` (the Primitive `cause` carries the cap).

**Errors:** `querying` — `invalid_query` (malformed id, or batch exceeds the implementation's cap; absence is `null` per position, not an error).

**Access:** `query`.

**Concurrency:** concurrent-safe.

---

### `list_scope`

```typescript
function list_scope(
  scope_path:        ScopePath,
  status_filter?:    AtomStatus[],
  include_derived?:  boolean,             // default false
  page?:             PageRequest
): Promise<Page<Atom>>
```

Atoms at one node of the projection scope tree. `status_filter` defaults to present states only (`["active", "hanging", "deferred", "deprecated"]`); pass the removal states to include tombstones, which are present until purged by garbage collection.

`include_derived` defaults to `false` — derived atoms are hidden from listings by default; consumers explicitly opt in.

Paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `invalid_query` (bad scope path, or invalid paging cursor).

**Access:** `query`.

---

### `list_children`

```typescript
function list_children(scope_path: ScopePath): Promise<ScopePath[]>
```

Child scopes of a projection-scope tree node.

**Errors:** `querying` — `invalid_query` (bad scope path).

**Access:** `query`.

---

### `list_by_classification`

```typescript
function list_by_classification(
  classification_id:  AtomId,                // ClassificationAtom
  options?: {
    tree?:            TreeId                 // restrict to one tree
    include_descendant_classifications?: boolean  // default true — also includes atoms classified under sub-classifications
    status_filter?:   AtomStatus[]
    include_derived?: boolean
  },
  page?:              PageRequest
): Promise<Page<Atom>>
```

Returns all atoms whose `is_a` chain (walked to root) passes through the given ClassificationAtom. With `include_descendant_classifications: true`, includes atoms classified under any sub-classification of the given one. Paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `not_found` (unknown classification id), `invalid_query` (unknown tree in `options.tree`, or invalid paging cursor).

**Access:** `query`.

---

### `list_by_tree`

```typescript
function list_by_tree(
  tree:              TreeId,
  status_filter?:    AtomStatus[],
  include_derived?:  boolean,
  page?:             PageRequest
): Promise<Page<Atom>>
```

All atoms in the given tree — the granularity at which a knowledge base grows without bound, so always paged ([00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads)). Used for cross-tree analyses (similarity, equivalence candidates) and for tree-level audits.

**Errors:** `querying` — `invalid_query` (unknown tree, or invalid paging cursor).

**Access:** `query`.

---

### `resolve_subject`

```typescript
function resolve_subject(
  surface_form:  string,                     // prose name to resolve
  tree:          TreeId,
  hint?: {
    expected_classification?: AtomId         // narrow the resolution to a classification
  }
): Promise<ResolveResult>

interface ResolveResult {
  matches: ResolvedMatch[]
  best?:   AtomId                            // populated when there's a confident single match
}

interface ResolvedMatch {
  atom_id:        AtomId
  via:            "exact" | "alias" | "fuzzy"
  alias_source?:  AtomId                     // ClassificationAtom whose aliases produced this match
  confidence:     number                     // 0.0–1.0
}
```

Resolves a prose surface form to one or more AtomIds via the within-tree alias graph. Used by extractors when promoting prose to AtomId references.

**Errors:** `querying` — `invalid_query` (unknown tree).

**Access:** `query`.

---

### `traverse_relations`

```typescript
function traverse_relations(
  atom_id:    AtomId,
  options?: {
    direction?:        "outgoing" | "incoming"   // default outgoing
    kind_filter?:      PredicateKind[]           // restrict to specific predicate kinds
    predicate_filter?: AtomId[]                  // restrict to specific PredicateAtoms
    include_derived?:  boolean                   // default false; true when the traversal is filtered entirely to the aggregation family — see below
    depth_limit?:      number                    // default 1 (immediate neighbors only)
    tree_filter?:      TreeId[]                  // restrict to specific trees
  },
  page?:      PageRequest
): Promise<Page<EdgeRecord>>

interface EdgeRecord {
  relation_id:    AtomId                          // the RelationAtom
  predicate_id:   AtomId                          // its PredicateAtom (via is_a)
  predicate_kind: PredicateKind
  source:         AtomId
  target:         AtomId
  is_derived:     boolean
}
```

Walks the RelationAtom graph from one atom. `outgoing` means follow edges where the atom is the source; `incoming` means follow edges where the atom is the target.

Filters compose (intersected). `depth_limit > 1` performs multi-hop traversal. The enumeration pages the **edge sequence** of the full filtered traversal, in the implementation's stable traversal order ([00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads)); the set of atoms reached is recoverable as the union of `source` / `target` over the enumerated edges (plus the origin). Hydrate reached atoms with [`get_by_ids`](#get_by_ids).

**Derived-edge default.** `include_derived` defaults to `false`, except when the traversal is filtered entirely to the aggregation family — every `kind_filter` entry is `aggregation` or `member_of`, or every `predicate_filter` entry resolves to a predicate whose kind is in the aggregation family — where it defaults to `true`: containment queries see chain-derived containment without opt-in, per [03 — Data Model § Chain derivation](../spec/03-data-model.md#chain-derivation). Pass `include_derived: false` explicitly to restrict such a traversal to asserted structure.

**Errors:** `querying` — `not_found` (unknown origin atom), `invalid_query` (unknown tree in `tree_filter`, or invalid paging cursor).

**Access:** `query`.

---

### `simulate_transition`

```typescript
function simulate_transition(
  atom_id:       AtomId,
  hypothetical:  AtomStatus,             // the state to evaluate the cascade for
  options?:      { depth_limit?: number }
): Promise<CascadeSimulation>

interface CascadeSimulation {
  origin:        AtomId
  origin_state:  AtomStatus
  impacts:       SimulatedImpact[]       // every atom the cascade fixpoint would transition
  derived_loss:  AtomId[]                // derived atoms that would be invalidated
}

interface SimulatedImpact {
  atom_id:         AtomId
  predicted_state: AtomStatus
  via:             CascadeStep[]         // path(s) from the origin — shared shape (00-shared-types.md § Cascade paths)
}
```

Read-only **dry-run of the cascade engine** (see [07 — Atom Lifecycle](../spec/07-atom-lifecycle.md)): computes the full three-channel cascade fixpoint — predicate-kind rules, scope-premise (`constraint_atoms`), derivation-invalidation, with the composition confidence gate applied — for a hypothetical transition, **without mutating any atom**. It is the *same* engine the Status Manager runs when it actually applies a transition. Cascade paths are **real-edged**: the engine walks asserted RelationAtoms only ([07 § Cascade rules](../spec/07-atom-lifecycle.md#cascade-rules)) — every `via` step's `edge_chain` names asserted edges; derived edges appear in the result only as `derived_loss` (channel-3 invalidations), never as cascade transmitters. `kind=equivalence` links transmit no cascade and are never walked ([07 § Tree boundaries and cascade](../spec/07-atom-lifecycle.md#tree-boundaries-and-cascade)); [Blast Radius](./blast-radius.md) surfaces equivalence-linked twins separately, as advisory entries outside the structural impact set.

**Equivalence invariant:** the impact set returned by `simulate_transition(a, s)` MUST equal the set of atoms an actual `update_status(a, s, …)` would transition, with the same `predicted_state` per atom. The two are the dry-run and the real run of one engine and MUST NOT diverge. [Blast Radius](./blast-radius.md) obtains its structural impact from this method rather than re-implementing the traversal.

A hypothetical of `superseded` yields an **empty** structural impact set: succession is silent by the observable resolution invariant ([spec/07 § Observable resolution invariant](../spec/07-atom-lifecycle.md#observable-resolution-invariant)) — on an actual transition, references resolve to the (gate-guaranteed compatible, `active`) successor, so nothing hangs. Derived atoms premised on the origin still appear in `derived_loss`; the actual transition recomputes them against the successor.

**Errors:** `querying` — `not_found` (unknown atom).

**Access:** `query`.

**Concurrency:** concurrent-safe (read-only).

---

### `get_classification_schema`

```typescript
function get_classification_schema(
  classification_id:  AtomId
): Promise<ResolvedSchema>

interface ResolvedSchema {
  classification_id:        AtomId
  subject_kind?:            "named" | "referential"   // absent if abstract (not yet fixed in the chain)
  domain_classifications?:  AtomId[]                   // referential only: allowed classes of the subject reference
  attributes:               AttributeSchema[]          // accumulated from ancestors
  required_relations:       RelationConstraint[]       // accumulated from ancestors
  optional_relations:       RelationConstraint[]       // accumulated from ancestors
  disjoint_with:            AtomId[]                   // accumulated from ancestors
  max_children:             number
  is_a_chain:               AtomId[]                   // walked from this classification to root
}
```

Returns the schema for a ClassificationAtom, resolving inheritance from ancestor classifications (`subject_kind` resolves to the point it was fixed in the chain; attributes/required_relations/optional_relations/disjoint_with accumulate).

Callers use this to know what attributes and edges atoms classified under this ClassificationAtom must carry.

**Errors:** `querying` — `not_found` (unknown classification id).

**Access:** `query`.

---

### `get_predicate_schema`

```typescript
function get_predicate_schema(
  predicate_id: AtomId
): Promise<ResolvedPredicateSchema>

interface ResolvedPredicateSchema {
  predicate_id:           AtomId
  kind:                   PredicateKind
  hard_characteristics:   PropertyCharacteristics // from PredicateKindAtom
  effective_characteristics: PropertyCharacteristics // hard + soft (from this PredicateAtom and ancestors)
  domain_classifications: AtomId[]                  // accumulated from sub-predicate hierarchy
  range_classifications:  AtomId[]                  // accumulated
  aliases:                string[]
  inverse_of?:            AtomId
  chain?:                 ChainStep[]               // as declared on this PredicateAtom; chains are not inherited through the sub-predicate hierarchy
  is_a_chain:             AtomId[]                  // walked to PredicateKindAtom
}
```

Returns the resolved schema for a PredicateAtom, walking the sub-predicate hierarchy to the PredicateKindAtom.

**Errors:** `querying` — `not_found` (unknown predicate id).

**Access:** `query`.

---

### `changes_since`

Conforms to the shared signature from [`00-shared-types.md`](./00-shared-types.md). Event variants:

```typescript
type AtomsEvent =
  | { kind: "Added";                       payload: { atom_id: AtomId; atom: Atom; outcome_kind?: ConflictOutcomeKind } }   // outcome_kind present iff the add is a conflict-pipeline effect — a materialized resolution edge or a conflicts_with linkage record (spec/06 § Outcome recording)
  | { kind: "DerivedAdded";                payload: { atom_id: AtomId; atom: Atom; derivation: Derivation } }   // advisory — records a derived atom's materialization; applies as a no-op to replayed asserted state (spec/05 § Derived atoms in the stream)
  | { kind: "Updated";                     payload: { atom_id: AtomId; before: Atom; after: Atom; outcome_kind?: ConflictOutcomeKind } }   // in-place content revision (spec/03 § Mutability and revision); never a status change, never cascaded; outcome_kind present iff pipeline-applied (a revision-class resolution, or a conflict record written to annotations)
  | { kind: "StatusChanged";               payload: {
        atom_id:          AtomId;
        from:             AtomStatus;
        to:               AtomStatus;
        rationale:        string;                                // human-readable; informative only — consumers MUST NOT parse it
        cascade_channel?: "predicate-kind" | "scope-premise";    // present iff the transition was cascaded
        cause_atom_id?:   AtomId;                                // present iff cascaded — the upstream triggering atom (channel 1) or the gating ConstraintAtom (channel 2)
        predicate_kind?:  PredicateKind;                         // present iff cascade_channel = "predicate-kind"
        outcome_kind?:    ConflictOutcomeKind;                   // present iff a conflict-pipeline resolution drove the transition (spec/06 § Outcome recording)
    } }
  | { kind: "Superseded";                  payload: { atom_id: AtomId; superseded_by: AtomId; rationale: string; outcome_kind?: ConflictOutcomeKind } }
  | { kind: "Duplicate";                   payload: { atom_id: AtomId; duplicate_of: AtomId; rationale: string; outcome_kind?: ConflictOutcomeKind } }
  | { kind: "Rejected";                    payload: { atom_id: AtomId; rationale: string; outcome_kind?: ConflictOutcomeKind; candidate?: Atom; involved?: AtomId[] } }   // candidate present iff this records the pipeline's rejection of a never-committed candidate (hard-block or non-entering fold) — the audit record; involved names implicated existing atoms
  | { kind: "Removed";                     payload: { atom_id: AtomId; rationale: string; outcome_kind?: ConflictOutcomeKind } }
  | { kind: "Purged";                      payload: { atom_id: AtomId; rationale: string } }   // physical deletion of a tombstone by GC (system-correlated)
  | { kind: "DerivedInvalidated";          payload: { atom_id: AtomId; source: AtomId; rule: string } }   // advisory — channel-3 invalidation of a materialized derived atom (spec/07 § Channel 3); applies as a no-op to replayed asserted state (spec/05 § Derived atoms in the stream)
  | { kind: "CompositionCascadeSkipped";   payload: { relation_id: AtomId; cause_atom_id: AtomId; spared_atom_id: AtomId; confidence: number; threshold: number } }   // the composition confidence gate suppressed a whole→part cascade — the spared part stays active (spec/07 § Composition cascade — confidence gate); advisory: applies as a no-op to materialized state
  | { kind: "ClassificationSchemaUpdated"; payload: { classification_id: AtomId; before: ClassificationSchema; after: ClassificationSchema } }
  | { kind: "PredicateSchemaUpdated";      payload: { predicate_id: AtomId; before: PredicateSchema; after: PredicateSchema } }
```

There are no dedicated conflict-outcome event kinds: outcome effects emit the ordinary events above, carrying the `outcome_kind` classification in their payloads ([06 § Outcome recording](../spec/06-conflict-classification.md#outcome-recording)).

**Ordering:** events within the stream are totally ordered. Causality rules from [05 — Delta Streams](../spec/05-delta-streams.md) apply: `Added(A)` precedes any event referencing `A`; `Updated(A)` follows `Added(A)`, in revision order; `DerivedAdded(D)` follows `Added(S)` for every `S` in `D.derivation.from`; `DerivedInvalidated(D)` follows the source-atom transition (or the `Updated` event, for revision-triggered recomputes); `CompositionCascadeSkipped` follows the source-atom transition whose cascade run it records and applies as a **no-op** to materialized state (advisory only). `DerivedAdded` and `DerivedInvalidated` are likewise advisory — materialization records that apply as no-ops to replayed asserted state; consumers recompute derived state instead of consuming it ([05 § Derived atoms in the stream](../spec/05-delta-streams.md#derived-atoms-in-the-stream)). Sole exception to the `Added`-first rule: a `Rejected` event whose payload carries `candidate` records the rejection of a never-committed candidate — its `atom_id` has no prior `Added`, and the event applies as a **no-op** to materialized state (audit only; [05 § Causal ordering](../spec/05-delta-streams.md#causal-ordering-within-a-stream)).

**Cascade provenance:** which transitions carry the `cascade_channel` / `cause_atom_id` / `predicate_kind` fields, and their values per channel, are defined in [07 § Cascade provenance emissions](../spec/07-atom-lifecycle.md#cascade-provenance-emissions). Channel-3 invalidations emit `DerivedInvalidated`, not `StatusChanged`.

`snapshot(ref_X) + events(ref_X → ref_Y) ≡ snapshot(ref_Y)`.

---

## Version notes

- **`aala-v0.1`** — initial contract; pre-publish.
- Backward-compatible additions allowed: new optional fields in `Atom`, new methods, new ConflictOutcomeKind variants (consumers must tolerate), new event kinds (consumers must tolerate as no-ops), new standard-library ClassificationAtoms and PredicateAtoms.
- Breaking changes (bump major): adding/removing concrete AtomType values, adding/removing PredicateKind values, changing cascade semantics, changing schema-inheritance rules, removing or repurposing existing variants, changing required parameters, changing idempotency rules.

## Notes for implementers

- The Status Manager is the only path that mutates the Atom Store. Conflict's classification work, all three cascade channels, derivation invalidation, the `update_status` call, and the `update` content revision (direct or pipeline-applied) all funnel through it. This is enforced by the API-only access rule ([02 — Conformance](../spec/02-conformance.md#api-only-access)).
- The within-tree alias graph and the cross-tree similarity index are not part of the public interface. Implementations choose techniques (string match, embedding NN, hybrid).
- Derived-atom materialization (eager / lazy / hybrid) is an implementation choice, and the derived-atom events follow it: `DerivedAdded` / `DerivedInvalidated` are advisory records of materialization activity — a lazily caching implementation emits none, and consumers never depend on them ([05 § Derived atoms in the stream](../spec/05-delta-streams.md#derived-atoms-in-the-stream)). When emitted, `DerivedAdded(D)` follows `Added(S)` for every `S` in `D.derivation.from`.
- Removal outcomes (`superseded` / `duplicate` / `rejected` / `removed`) are applied as committed tombstone transitions — the atom is retained with the removal-outcome status. Physical deletion occurs only via an optional, `system:`-correlated garbage-collection pass that emits `Purged` (see [07 — Atom Lifecycle](../spec/07-atom-lifecycle.md) § Garbage collection).
