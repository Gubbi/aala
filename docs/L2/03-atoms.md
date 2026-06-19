# Atoms — The Canonical Knowledge Record

**Status: non-optional.** Every aala deployment includes this container; everything else either feeds it or reads from it.

## Concern

Atoms is the System of Record. It owns the canonical typed grammar for claims, the durable representation of every atom, and every operation that mutates the canonical record.

In one sentence: **Atoms knows what an atom is, what it's classified under, how it relates to other atoms, how it was born, how it changes, and what consequences entail from it.**

Atoms also implements the [Projection facet](./04-projection.md) (required) over its own content — exposing canonical claim prose and the glossary, plus a navigable index — as a second read surface alongside the structured read API below. External agents compose answers and documents from these surfaces; see [`analysis/agent-integration-pattern.md`](../../analysis/agent-integration-pattern.md).

## The typed grammar

Five concrete atom types form the core grammar (see [`spec/03-data-model.md`](../../spec/03-data-model.md) for normative details):

| Type | Role |
|---|---|
| `entity` | Ordinary subjects — things the snapshot makes claims about. Decisions, constraints, behaviors, capabilities, scenarios, people, organizations are all entities classified under standard-library ClassificationAtoms. |
| `relation` | Single triples between two atoms. The only atom type carrying a `subject/object` edge. |
| `classification` | Classes of subjects with structural schemas. The "what does it mean to be a kind of X?" layer. |
| `predicate` | Named predicates with domain/range/characteristics. Specialized classifications whose instances are RelationAtoms. |
| `predicate_kind` | The ten pre-registered kinds — four inverse pairs (`specialization`⟷`generalization`, `aggregation`⟷`member_of`, `composition`⟷`component_of`, `dependency`⟷`enablement`) plus the self-inverse `association` and `equivalence` — carrying intrinsic cascade semantics and hard characteristics. |

The grammar replaces the older notion of "one atom type per claim flavor" (DecisionAtom, ConstraintAtom, etc.) with a uniform `entity` type classified under ClassificationAtoms. The standard library pre-registers the familiar claim classifications so callers don't lose ergonomic access.

## What Atoms owns

| Concern | Notes |
|---|---|
| Atom schema | The base shape and per-type rules. Field requirements, status enum, scope structure. The schema is owned here because Extraction lives here — the producer of a record owns its shape. |
| Standard-library registration | Per tree, the pre-registered ClassificationAtoms and PredicateKindAtoms (see [`spec/03`](../../spec/03-data-model.md)). Atoms initializes the standard library at tree creation. |
| Durable storage | Atoms persisted in a diffable form. Persistence is an implementation choice; the conceptual contract is durability + diff-friendliness. |
| Extraction | Fragments → proposed atoms. Extractors emit `entity`, `relation`, `classification`, and `predicate` atoms; the standard-library lookup resolves their classifications and predicates. |
| Conflict classification | New atoms classified against the canonical record. Twenty-plus structured outcomes with three resolution modes (auto-resolve / soft-prompt / hard-block). See [`spec/06`](../../spec/06-conflict-classification.md). |
| Status transitions | Present states (`active`/`hanging`/`deferred`/`deprecated`) and removal outcomes (`superseded`/`duplicate`/`rejected`/`removed`). Status Manager is the single funnel. |
| Cascade machinery | Three channels: predicate-kind (dependency / composition rules), scope-premise (`Scope.constraint_atoms` reverse-index), derivation-invalidation. Run to fixpoint on every status transition. See [`spec/07`](../../spec/07-atom-lifecycle.md). |
| Relation traversal | Walks the RelationAtom graph filtered by predicate kind / specific PredicateAtom / depth / tree. Replaces the older references/depends_on graph. |
| Classification resolution | Walks `is_a` chains, accumulates schemas (attributes + required_relations + disjoint_with) from ancestors, validates proposed atoms against the cumulative schema. |
| Predicate resolution | Walks PredicateAtom hierarchies to the underlying PredicateKindAtom, accumulating characteristics and domain/range constraints. |
| Entailment computation | All three tiers (T1 transitive `is_a` + inverse_of + alias closure; T2 transitive composition/dependency/equivalence + symmetric expansion + cross-tree equivalence; T3 disjointness propagation + functional violations). Caching strategy is an implementation choice. |
| Alias graph | Within-tree alias index used by extraction to resolve prose surface forms to canonical AtomIds. |
| Cross-tree similarity | An advisory index that surfaces `CrossTreeEquivalenceCandidate` outcomes — never auto-merges. |

Conflict classification operates entirely on atom state and the schemas resolved from classifications/predicates. Its internal indices (deterministic match, embedding NN, alias graph) exist to make classification fast; consumers needing their own indexes build them by walking the public read APIs.

## Cascade triggered by Conflict

When Conflict classifies a proposal that mutates canonical state, the Status Manager applies the change and runs the cascade fixpoint across three independent channels:

1. **Predicate-kind cascade** — for each RelationAtom incident on the changing atom, apply the kind's cascade rule (dependency: target Hanging/Removed → source Hanging; composition: source Removed → target Hanging above confidence threshold).
2. **Scope-premise cascade** — every atom whose `Scope.constraint_atoms` contains the changing atom's id transitions from `active` to `hanging`.
3. **Derivation invalidation** — every derived atom whose `derivation.from` contains the changing atom is invalidated and either recomputed or removed.

The fixpoint is the floor case of impact propagation. [Blast Radius](./07-blast-radius.md), when wired, surfaces the same propagation as a queryable read-only report — useful for previewing impact before committing a transition.

## Trees

Atoms are tree-bounded via `Scope.tree`. Within a tree, same-claim atoms collapse (via `Duplicate` outcome or alias merge). Across trees, parallel claims can coexist linked by `RelationAtom(predicate kind=equivalence)`. The tree registry is owned by Orchestration; Atoms enforces tree-scoped Duplicate detection, disjointness checks, and classification chains.

The standard library is pre-registered per tree — each tree initializes its own classification hierarchy + predicate kinds.

## API surface (conceptual)

The public read API is structured around atoms, classifications, predicates, and relations. Atoms does not expose ad-hoc by-field query APIs; callers that need indexed lookups build their own indexes by walking the public read API.

**Read:**
- `get_by_id(atom_id)` — direct lookup.
- `list_scope(scope_path, options?)` — atoms at one node of the projection scope tree.
- `list_children(scope_path)` — child scopes.
- `list_by_classification(classification_id, options?)` — atoms whose `is_a` chain passes through the given ClassificationAtom (optionally including sub-classifications).
- `list_by_tree(tree, options?)` — all atoms in a tree.
- `resolve_subject(surface_form, tree, hint?)` — prose → AtomId via the alias graph.
- `traverse_relations(atom_id, options?)` — walk the RelationAtom graph; filterable by predicate kind, specific PredicateAtom, depth, tree.
- `get_classification_schema(classification_id)` — resolved schema with inheritance accumulated.
- `get_predicate_schema(predicate_id)` — resolved schema with sub-predicate inheritance.
- `changes_since(ref)` — ordered delta stream. Variants include `Added`, `DerivedAdded`, `Updated`, `StatusChanged`, `Superseded`, `Duplicate`, `Rejected`, `Removed`, `DerivedInvalidated`, `ConflictOutcomeEmitted`, `ConflictOutcomeResolved`, `ClassificationSchemaUpdated`, `PredicateSchemaUpdated`.

**Write (against the currently selected snapshot):**
- `propose(fragment, hints?)` — runs extraction + conflict classification; produces a `ProposeResult` enumerating added atoms, derived atoms, per-atom conflict outcomes (with mode and resolution paths), and cascade records.
- `update_status(atom_id, outcome, rationale, meta?)` — the single mutation API outside `propose`. Status Manager runs the full cascade fixpoint before returning.

Atoms operates against whichever snapshot is currently selected. Snapshot lifecycle (fork / switch / publish / discard) is owned by Orchestration. Concrete snapshot realizations (git working tree, versioning DB, content-addressed manifest) are implementation choices.

The binding interface contract — exact signatures, error model, idempotency rules — is in [`interfaces/atoms.md`](../../interfaces/atoms.md).

## Dependencies

- **[LLM Gateway](./09-llm-gateway.md)** *(non-optional infrastructure)* — for Extraction (`aala.extraction`), Conflict's structural judges (`aala.conflict_judge`), classification suggestions (`aala.classification_suggest`), predicate suggestions (`aala.predicate_suggest`).
- **[Ingestion](./02-ingestion.md)** *(non-optional)* — produces the normalized fragments that Extraction consumes.

No dependencies on optional containers (per the architecture principle).

## Components (preview of L3)

- **Schema** — atom base + per-type rules; validator that walks classification chains.
- **Atom Store** — durable persistence (implementation-specific).
- **Standard Library** — pre-registered ClassificationAtoms and PredicateKindAtoms per tree.
- **Extraction Router** — picks extractors per content-kind.
- **Extractors** — pluggable: entity extractors (per standard-library classification), relation extractor, classification extractor, predicate extractor.
- **Conflict Pipeline** — multi-stage: structural validation → alias resolution → schema validation → duplicate / similarity → edge-specific checks → equivalence / alias coherence → capacity / promotion → cross-tree advisory.
- **Alias Graph** — within-tree index supporting `resolve_subject`.
- **Embedding Index** — internal to Conflict; not exposed externally.
- **Cross-tree Similarity Index** — surfaces `CrossTreeEquivalenceCandidate` outcomes.
- **Schema Resolver** — walks `is_a` chains for classifications, sub-predicate chains for predicates; caches resolved schemas.
- **Entailment Engine** — computes T1/T2/T3; maintains derived-atom materialization (lazy/eager/hybrid per implementation choice); invalidates derivations on source changes.
- **Status Manager** — the single funnel for canonical mutation. Applies transitions and runs the three-channel cascade fixpoint.
- **Change Log** — ordered, append-only event log; serves `changes_since(ref)` and manages checkpoint refs for downstream containers.

Full L3 detail lives in [`docs/L3/03-atoms.md`](../L3/03-atoms.md).

## Variation points (where implementations differ)

| Variation | Examples |
|---|---|
| Storage backend | YAML in git, Postgres + git-mirror, object-store-backed. |
| Extractor set | Minimal (entity + relation only) vs full (with classification suggestion, predicate suggestion). |
| Conflict pipeline depth | Structural-only (fast, low recall) → +NN similarity → +LLM judge for ambiguous outcomes. |
| Entailment caching | Lazy (compute on query) / Eager (materialize on write) / Hybrid (cache hot derivations). |
| Embedding model | Tied to LLM Gateway's routing; per-implementation trade-off between dim, latency, recall. |
| Standard library extensions | Deployments may add classifications and predicate atoms beyond the standard library (e.g., `Classification("Compliance-Constraint")` derived from `Classification("Constraint")`). The standard library is the floor; deployments narrow or extend. |
| Cross-tree similarity strategy | Deterministic-only / embedding-driven / hybrid; controls precision/recall on `CrossTreeEquivalenceCandidate`. |
