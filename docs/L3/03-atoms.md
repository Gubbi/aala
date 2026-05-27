# L3 — Atoms Components

For the container framing, see [`L2/03-atoms.md`](../L2/03-atoms.md). Atoms owns the canonical schema for claims, the durable record, extraction, conflict detection, status transitions, alias / reference primitives, and the embedding-NN index. It is the largest L3 chapter because it absorbs Extraction and Conflict as primitives.

## Component diagram

![L3 — Atoms](../diagrams/atomsInternals.png)

## Component reference

| Component | Responsibility | Internal state | Emits / consumes |
|---|---|---|---|
| **Schema** | Defines atom shape and subtypes. Validates atoms on write. The single authority on what an atom is. | Schema versions (declared, not derived). | Used by Atom Store on write; by Extractors when shaping output. |
| **Atom Store** | Durable persistence of atoms in the current snapshot. Backend is implementation-specific. | The atoms themselves. | Receives writes from Status Manager (status) and Extraction (new). |
| **Extraction Router** | Picks which Extractor(s) handle a fragment based on its content kind. | Router config. | In: `NormalizedFragment` from caller. Out: routes to one or more Extractors. |
| **Extractors** | One per content kind: decision, definition, dependency, generic. Produces proposed atoms with provenance fields populated. | Prompt templates / patterns (impl-specific). | Calls LLM Gateway with `aala.extraction`. Out: `ProposedAtom[]`. |
| **Conflict Pipeline** | Classifies each proposed atom against canonical. Three stages: deterministic match → embedding NN → LLM judge. | Per-atom classification history. | Reads from Embedding Index + Atom Store. Calls LLM Gateway with `aala.conflict_judge`. Out: classifications. |
| **Embedding Index** | Internal NN store used only by Conflict's candidate-finding stage. Not exposed externally. | Atom embeddings keyed by atom id. | Calls LLM Gateway with `aala.embedding_default`. |
| **Status Manager** | The single mutation funnel for canonical atom state. Applies present-state transitions (`Active` / `Hanging` / `Deferred` / `Deprecated`) and removal outcomes (`Superseded` / `Duplicate` / `Rejected` / `Removed`, each with intent meta — `superseded_by`, `duplicate_of`, rationale). Drives the Hanging cascade via direct references after Conflict applies Supersedes / Refines. | None of its own (state lives on atoms). | Reads atoms; writes status or applies removal. Triggers Change Log `StatusChanged` / `Superseded` / `Duplicate` / `Rejected` / `Removed` events with full intent meta. |
| **Lookup APIs** | Serves read primitives: `get_by_id`, `list_scope`, `list_children`, `traverse_references`. | None. | Pure reads against Atom Store. |
| **Change Log** | Maintains the ordered, append-only event log for the container. | Event sequence + ref / checkpoint surface. | Emits `Added` / `Updated` / `StatusChanged` / `Deleted`. Serves `changes_since(ref)` for [Hierarchical Navigation](./06-hierarchical-nav.md), [Projection](./04-projection.md), [Blast Radius](./07-blast-radius.md), [Quality](./10-quality.md). |

## Internal flow — proposal pipeline

```mermaid
sequenceDiagram
    participant Caller as Caller (e.g., Orchestration)
    participant Router as Extraction Router
    participant Extr as Extractors
    participant LLM as LLM Gateway
    participant Conf as Conflict Pipeline
    participant Embed as Embedding Index
    participant Lookup as Lookup APIs
    participant Status as Status Manager
    participant Store as Atom Store
    participant Log as Change Log

    Caller->>Router: propose(fragment)
    Router->>Extr: route by content kind
    Extr->>LLM: aala.extraction
    LLM-->>Extr: ProposedAtom[]
    Extr->>Conf: classify(proposed_atoms)
    Conf->>Embed: NN search per atom
    Conf->>Lookup: read candidate atoms
    Conf->>LLM: aala.conflict_judge (top-tier)
    LLM-->>Conf: classifications
    Conf->>Status: apply outcomes (Add / Superseded / Duplicate / Rejected / ...)
    Status->>Store: write atoms + apply removal outcomes
    Status->>Store: cascade Hanging via direct refs
    Store->>Log: Added / Updated / StatusChanged / Superseded / Duplicate / Rejected / Removed
    Log-->>Caller: ingest_id, classification summary
```

## Canonical data model

The schema below is the conceptual data model. Specific implementations may serialize it differently (YAML, JSON, protobuf, …). The interface contract for exact shape lives in `docs/interfaces/atoms.md`.

```mermaid
classDiagram
    class Atom {
        +String id
        +AtomType type
        +String subject
        +String predicate
        +String object
        +Map attributes
        +AtomStatus status
        +String status_reason
        +Datetime status_changed_at
        +List~Premise~ premises
        +List~Source~ sources
        +Float confidence
        +Map confidence_factors
        +Scope scope
        +List~String~ references
        +List~String~ depends_on
    }

    class DecisionAtom {
        +String rationale
        +List~String~ supersedes
        +DecisionMetadata decision_metadata
    }

    class DefinitionAtom {
        +List~String~ aliases
        +List~String~ see_also
        +String canonical_for
    }

    class Source {
        +String uri
        +String quote
        +Datetime extracted_at
    }

    class Premise {
        +String key
        +String value
    }

    class Scope {
        +String environment
        +TimeScope time_scope
        +List~String~ conditions
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
        dependency
        decision
        constraint
        behavior
        capability
        definition
        scenario
    }

    Atom <|-- DecisionAtom
    Atom <|-- DefinitionAtom
    Atom "1" *-- "0..*" Source
    Atom "1" *-- "0..*" Premise
    Atom "1" *-- "1" Scope
    Scope "1" *-- "1" TimeScope
    Atom --> AtomStatus
    Atom --> AtomType
```

The atom lifecycle state machine (covering the `AtomStatus` transitions) is in [`L2/11-flows.md`](../L2/11-flows.md#atom-lifecycle-state-machine).

## Variation points

| Variation | Examples |
|---|---|
| Storage backend | YAML files in git (local-first); relational DB rows; object store + manifest. |
| Extractor set | Minimal (decision + definition + generic); full (+ dependency, capability, scenario, behavior, constraint). |
| Conflict pipeline depth | Deterministic-only (fast, low recall); +NN (medium); +LLM judge (highest fidelity). |
| Embedding model | Selected by the deployer's gateway config under `aala.embedding_default` use-case key. |
| Schema extensions | Custom atom subtypes for tenant-specific domains. The global schema defines the floor, not the ceiling. |
