# L3 — Atoms Components

For the container framing, see [`L2/03-atoms.md`](../L2/03-atoms.md). Atoms owns the typed grammar for claims, the durable record, extraction, conflict classification, status transitions, cascade machinery, relation traversal, classification and predicate resolution, the alias graph, the cross-tree similarity index, and entailment computation.

It is the largest L3 chapter because it absorbs Extraction, Conflict, the Schema Resolver, the Entailment Engine, and the Status Manager as components inside one container. Atoms also implements the [Projection facet](./04-projection.md) (required) over its claims — canonical claim prose, the glossary, and a navigable index — as a second read surface alongside its structured read API.

## Component diagram

![L3 — Atoms](../diagrams/atomsInternals.png)

## Component reference

| Component | Responsibility | Internal state | Emits / consumes |
|---|---|---|---|
| **Schema** | Defines atom base shape and per-type rules. Validates atoms on write against the resolved classification schema (walked via Schema Resolver). | Schema versions (declared, not derived). | Used by Atom Store on write; by Extractors when shaping output. |
| **Atom Store** | Durable persistence of atoms in the current snapshot. Backend is implementation-specific. | The atoms themselves (asserted + derived). | Receives writes from Status Manager (status), Extractors (new asserted), Entailment Engine (new derived). |
| **Standard Library** | Pre-registered ClassificationAtoms and PredicateKindAtoms per tree. Initialized on tree creation; immutable as a baseline (deployments add but do not remove). | The pre-registered atoms per tree. | Read by Extractors, Schema Resolver, Conflict Pipeline, Validators. |
| **Extraction Router** | Picks which Extractor(s) handle a fragment based on its content kind and optional caller hints. | Router config. | In: `NormalizedFragment` from caller. Out: routes to one or more Extractors. |
| **Extractors** | Produce proposed atoms with provenance fields populated. Emit entity / relation / classification / predicate atoms. May call ClassificationSuggester or PredicateSuggester for novel terms. | Prompt templates / patterns (impl-specific). | Calls LLM Gateway with `aala.extraction`. Out: `ProposedAtom[]`. |
| **Classification Suggester** | For novel atoms without explicit classification, suggests a ClassificationAtom + is_a chain. | None. | Calls LLM Gateway with `aala.classification_suggest`. Out: suggested classification edge. |
| **Predicate Suggester** | For novel relationship surface forms, suggests a PredicateAtom + underlying PredicateKindAtom. | None. | Calls LLM Gateway with `aala.predicate_suggest`. Out: suggested PredicateAtom. |
| **Alias Graph** | Within-tree index supporting `resolve_subject` (prose → AtomId). Built from ClassificationAtom / PredicateAtom aliases plus EntityAtom names. | Per-tree alias index. | Receives updates from Atom Store via Change Log; queried by Conflict Pipeline (alias-resolution stage) and external `resolve_subject` calls. |
| **Schema Resolver** | Walks `is_a` chains for classifications and sub-predicate chains for predicates. Accumulates schema fields (attributes, required_relations, disjoint_with) and resolves `subject_kind` to the point it was fixed in the chain. Caches resolved schemas keyed by atom id. | Resolved-schema cache. | Reads Atom Store; cache invalidated by `ClassificationSchemaUpdated` / `PredicateSchemaUpdated` events. |
| **Conflict Pipeline** | 9-stage classification (structural validation → alias resolution → schema validation → duplicate / similarity → edge-specific structural → equivalence / alias coherence → capacity → promotion → cross-tree advisory). Produces structured outcomes with resolution modes. | Per-proposal classification context (ephemeral). | Reads Atom Store, Alias Graph, Schema Resolver, Embedding Index, Cross-Tree Similarity Index. Calls LLM Gateway with `aala.conflict_judge` for ambiguous cases. Out: outcomes. |
| **Embedding Index** | Internal NN store used by Conflict's similarity-detection stage. Not exposed externally. | Atom embeddings keyed by atom id. | Calls LLM Gateway with `aala.embedding_default`. |
| **Cross-Tree Similarity Index** | Surfaces `CrossTreeEquivalenceCandidate` outcomes by comparing new atoms against atoms in other trees. Advisory — never auto-merges. | Cross-tree embedding index. | Same embedding backbone as Embedding Index. |
| **Entailment Engine** | Computes T1 (transitive is_a, inverse_of materialization, alias closure), T2 (transitive composition/dependency/equivalence, symmetric expansion, cross-tree implications), T3 (disjointness propagation, functional violations producing conflict outcomes). Implementation chooses caching strategy (lazy / eager / hybrid). | Derived-atom materialization cache (impl-specific). | Reads Atom Store; writes derived atoms via Status Manager; emits `DerivedAdded` / `DerivedInvalidated` events. |
| **Status Manager** | The single mutation funnel for canonical atom state. Applies present-state transitions and removal outcomes. Runs the three-channel cascade fixpoint (predicate-kind, scope-premise, derivation-invalidation) on every transition. | None of its own (state lives on atoms). | Reads atoms; writes status. Triggers `ChangeEvent`s. Invokes Entailment Engine for derivation invalidation. |
| **Validators** | At proposal time: type-specific field requirements, subject treatment per classification's `subject_kind`, `is_a` target type per id-prefix rules (entity→classification, relation→predicate), schema-cumulative `required_relations`, predicate domain/range (and referential `domain_classifications`). | None. | Reads Schema Resolver outputs; raises `InvalidInput` or feeds outcomes into Conflict Pipeline. |
| **Lookup APIs** | Serves read primitives: `get_by_id`, `list_scope`, `list_children`, `list_by_classification`, `list_by_tree`, `resolve_subject`, `traverse_relations`, `get_classification_schema`, `get_predicate_schema`. | None. | Pure reads against Atom Store, Alias Graph, Schema Resolver. |
| **Change Log** | Maintains the ordered, append-only event log for the container. | Event sequence + ref / checkpoint surface. | Emits all atom events (Added, DerivedAdded, Updated, StatusChanged, Superseded, Duplicate, Rejected, Removed, Purged, DerivedInvalidated, ConflictOutcomeEmitted, ConflictOutcomeResolved, ClassificationSchemaUpdated, PredicateSchemaUpdated). Serves `changes_since(ref)` for [Hierarchical Navigation](./06-hierarchical-nav.md), [Blast Radius](./07-blast-radius.md), [Quality](./10-quality.md), and Atoms' own [Projection facet](./04-projection.md). |

## Internal flow — proposal pipeline

```mermaid
sequenceDiagram
    participant Caller as Caller (e.g., Orchestration)
    participant Router as Extraction Router
    participant Extr as Extractors
    participant ClsSug as Classification Suggester
    participant PredSug as Predicate Suggester
    participant LLM as LLM Gateway
    participant Vld as Validators
    participant Alias as Alias Graph
    participant Resv as Schema Resolver
    participant Conf as Conflict Pipeline
    participant Embed as Embedding Index
    participant XTree as Cross-Tree Similarity
    participant Status as Status Manager
    participant Store as Atom Store
    participant Ent as Entailment Engine
    participant Log as Change Log

    Caller->>Router: propose(fragment, hints)
    Router->>Extr: route by content kind
    Extr->>LLM: aala.extraction
    LLM-->>Extr: ProposedAtom[]
    opt classifications novel
        Extr->>ClsSug: suggest classification
        ClsSug->>LLM: aala.classification_suggest
        LLM-->>ClsSug: suggested is_a chain
    end
    opt predicates novel
        Extr->>PredSug: suggest predicate
        PredSug->>LLM: aala.predicate_suggest
        LLM-->>PredSug: suggested PredicateAtom
    end
    Extr->>Vld: validate per-type field requirements
    Vld->>Resv: walk is_a / sub-predicate chains
    Resv-->>Vld: resolved schema
    Vld-->>Conf: validated proposals
    Conf->>Alias: resolve prose subjects
    Conf->>Embed: NN similarity (within-tree)
    Conf->>XTree: NN similarity (cross-tree advisory)
    Conf->>Resv: schema-driven checks (required_relations, disjointness, domain/range)
    Conf->>LLM: aala.conflict_judge (ambiguous cases)
    LLM-->>Conf: classification + mode
    Conf->>Status: apply per outcome (Add / Supersede / hard-block / etc.)
    Status->>Store: write asserted atoms
    Status->>Status: run cascade fixpoint (3 channels)
    Status->>Ent: invalidate stale derivations + recompute
    Ent->>Store: write derived atoms (with derivation populated)
    Store->>Log: Added / DerivedAdded / StatusChanged / ConflictOutcomeEmitted / ...
    Log-->>Caller: ProposeResult (added, derived_added, outcomes, cascaded)
```

## Canonical data model

The schema below is the conceptual data model. Specific implementations may serialize it differently (YAML, JSON, protobuf, …). The exact interface shape lives in [`interfaces/00-shared-types.md`](../../interfaces/00-shared-types.md).

```mermaid
classDiagram
    class Atom {
        +String id
        +AtomType type
        +String subject
        +String is_a
        +Map attributes
        +Map annotations
        +List~Source~ sources
        +Float confidence
        +Scope scope
        +Status status
        +Derivation derivation
    }

    class Status {
        +AtomStatus state
        +CorrelationId correlation_id
        +String reason
        +Datetime changed_at
        +RemovalMeta meta
    }

    class EntityAtom {
    }

    class RelationAtom {
        +String object
    }

    class ClassificationAtom {
        +ClassificationSchema schema
    }

    class PredicateAtom {
        +PredicateSchema schema
    }

    class PredicateKindAtom {
        +PredicateKind kind
        +PredicateKindSemantics semantics
    }

    class ClassificationSchema {
        +SubjectKind subject_kind
        +List~String~ domain_classifications
        +List~AttributeSchema~ attributes
        +List~RelationConstraint~ required_relations
        +List~RelationConstraint~ optional_relations
        +Integer max_children
        +List~String~ aliases
        +List~String~ see_also
        +String canonical_for
        +List~String~ disjoint_with
        +String definition
    }

    class PredicateSchema {
        +SubjectKind subject_kind
        +List~AttributeSchema~ attributes
        +Integer max_children
        +List~String~ aliases
        +List~String~ see_also
        +List~String~ disjoint_with
        +List~String~ domain_classifications
        +List~String~ range_classifications
        +PropertyCharacteristics characteristics
        +String inverse_of
    }

    class PredicateKindSemantics {
        +PropertyCharacteristics hard_characteristics
        +CascadeRule cascade
        +List~String~ conflict_outcomes
    }

    class AttributeSchema {
        +String key
        +ValueType value_type
        +Boolean required
    }

    class RelationConstraint {
        +String predicate
        +String target_classification
        +Integer min
        +String max
    }

    class PropertyCharacteristics {
        +Boolean symmetric
        +Boolean asymmetric
        +Boolean irreflexive
        +Boolean transitive
        +Boolean functional
        +Boolean inverse_functional
    }

    class CascadeRule {
        +List~AtomStatus~ source_triggers_target_hanging
        +List~AtomStatus~ target_triggers_source_hanging
        +Float confidence_threshold
    }

    class Derivation {
        +List~String~ from
        +String rule
    }

    class Source {
        +String uri
        +String quote
        +Datetime extracted_at
    }

    class Scope {
        +String tree
        +String environment
        +TimeScope time_scope
        +List~String~ constraint_atoms
    }

    class TimeScope {
        +Date effective_from
        +Date effective_to
    }

    class AtomStatus {
        <<enumeration>>
        active
        hanging
        deferred
        deprecated
        superseded
        duplicate
        rejected
        removed
    }

    class AtomType {
        <<enumeration>>
        entity
        relation
        classification
        predicate
        predicate_kind
    }

    class PredicateKind {
        <<enumeration>>
        association
        aggregation
        composition
        generalization
        dependency
        equivalence
    }

    class SubjectKind {
        <<enumeration>>
        named
        referential
    }

    class ValueType {
        <<enumeration>>
        string
        number
        integer
        boolean
        date
        datetime
        duration
        uri
        structured
    }

    Atom <|-- EntityAtom
    Atom <|-- RelationAtom
    Atom <|-- ClassificationAtom
    Atom <|-- PredicateAtom
    Atom <|-- PredicateKindAtom
    ClassificationAtom "1" *-- "1" ClassificationSchema
    PredicateAtom "1" *-- "1" PredicateSchema
    PredicateKindAtom "1" *-- "1" PredicateKindSemantics
    PredicateKindSemantics "1" *-- "1" PropertyCharacteristics
    PredicateKindSemantics "1" *-- "1" CascadeRule
    PredicateSchema "1" *-- "1" PropertyCharacteristics
    ClassificationSchema "1" *-- "0..*" AttributeSchema
    ClassificationSchema "1" *-- "0..*" RelationConstraint
    PredicateSchema "1" *-- "0..*" AttributeSchema
    Atom "1" *-- "0..*" Source
    Atom "1" *-- "1" Scope
    Atom "0..1" *-- "0..1" Derivation
    Scope "1" *-- "0..1" TimeScope
    Atom --> AtomStatus
    Atom --> AtomType
    PredicateKindAtom --> PredicateKind
    ClassificationSchema --> SubjectKind
    AttributeSchema --> ValueType
```

The atom lifecycle state machine (covering the `AtomStatus` transitions) is in [`L2/11-flows.md`](../L2/11-flows.md#atom-lifecycle-state-machine).

## Variation points

| Variation | Examples |
|---|---|
| Storage backend | YAML files in git (local-first); relational DB rows; object store + manifest. |
| Extractor set | Minimal (generic entity + relation only); typical (+ classification suggester + predicate suggester); deployment-custom. |
| Conflict pipeline depth | Structural-only (fast, low recall); +NN similarity (medium); +LLM judge for ambiguous (highest fidelity). |
| Entailment caching | Lazy (compute at query); eager (materialize at write); hybrid (cache hot derivations). |
| Embedding model | Selected by the deployer's gateway config under `aala.embedding_default`. |
| Standard library extensions | Deployments add custom ClassificationAtoms / PredicateAtoms beyond the standard library. |
| Cross-tree similarity strategy | Deterministic-only / embedding-driven / hybrid; controls precision/recall on `CrossTreeEquivalenceCandidate`. |
| Schema-resolver caching | Per-snapshot cache invalidated by schema updates; per-axiom invalidation; no caching (always walk). |
