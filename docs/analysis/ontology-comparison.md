# Ontology / Knowledge-System Comparison

**Scope:** how the Knowledge Compiler atom model — five concrete atom types (`entity`, `relation`, `classification`, `predicate`, `predicate_kind`) under one typed grammar — stands against OWL DL, RDFS, SKOS, and labeled property graphs. The normative model is in [`spec/03-data-model.md`](../../spec/03-data-model.md).

**Positioning:** classical-ontology rigor comparable to OWL on most structural features (classes, sub-class and sub-predicate hierarchies, property characteristics, equivalence, disjointness, qualified existential/cardinality restrictions, domain/range, fixed-tier entailment), plus first-class workflow features OWL/RDFS/SKOS/PG lack (per-claim provenance, operation-level correlation, lifecycle states, snapshot-isolated versioning, tree-bounded parallel discourses, a structured conflict pipeline). It intentionally omits the deep formal-logic features (universal restrictions, class algebra, full FOL) that serve formal-logic domains rather than contestable organizational knowledge.

## Feature matrix

| Capability | OWL DL | RDFS | SKOS | Property graph | Ours |
|---|---|---|---|---|---|
| Classes / types | ✓ | ✓ | concepts | labels | ✓ — `AtomType` (5 concrete) + ClassificationAtom hierarchy |
| Subclass / generalization | ✓ | ✓ | broader / narrower | manual | ✓ — `is_a` (primary, single) + `kind=specialization` (multi); transitive closure auto-derived (T1) |
| Subproperty hierarchy | ✓ | ✓ | ✗ | ✗ | ✓ — PredicateAtoms form sub-predicate chains (`predicate.is_a → predicate`); the ten predicate *kinds* (inverse-paired cascade semantics) remain a closed enum |
| Properties as first-class | ✓ | ✓ | ✗ | partial | ✓ — `RelationAtom` (instances of `PredicateAtom`) |
| Property characteristics — symmetric | ✓ | ✗ | ✗ | ✗ | ✓ |
| Property characteristics — asymmetric | ✓ | ✗ | ✗ | ✗ | ✓ |
| Property characteristics — transitive | ✓ | ✗ | ✗ | ✗ | ✓ (metadata + auto-derived closure, T2; composition family deliberately non-transitive — containment closure via declared chains) |
| Property characteristics — functional | ✓ | ✗ | ✗ | ✗ | ✓ |
| Property characteristics — inverse_functional | ✓ | ✗ | ✗ | ✗ | ✓ (hard on `composition` — single-parent rule) |
| Property characteristics — irreflexive | ✓ | ✗ | ✗ | ✗ | ✓ |
| Property characteristics — reflexive | ✓ | ✗ | ✗ | ✗ | ✗ — deliberately omitted |
| Inverse pairs (`inverse_of`) | ✓ | partial | ✗ | ✗ | ✓ — auto-materialized (T1) |
| Equivalence | ✓ (single `equivalentClass`) | ✗ | exactMatch | ✗ | ✓✓ — three levels: `aliases` (names), `kind=equivalence` (atoms), `Duplicate` outcome (ingestion), with bright-line rules in [06](../../spec/06-conflict-classification.md) |
| Disjointness | ✓ | ✗ | ✗ | ✗ | ✓ — `disjoint_with` on ClassificationSchema (within-tree); propagation through `is_a` chains (T3) |
| Qualified cardinality restrictions | ✓ (min/max/exact) | ✗ | ✗ | ✗ | ✓ — `min`/`max` per `(predicate, target_classification)` in `required_relations` / `optional_relations`; plus `max_children` on a classification |
| Existential restrictions (∃P.C) | ✓ | ✗ | ✗ | ✗ | ✓ — `required_relations` with `min ≥ 1` to a `target_classification` (someValuesFrom-equivalent) |
| Universal restrictions (∀P.C) | ✓ | ✗ | ✗ | ✗ | ✗ — not modeled |
| Domain / range constraints | ✓ | ✓ | ✗ | ✗ | ✓ — `domain_classifications` / `range_classifications` on predicates; referential classifications constrain their subject reference via `domain_classifications`; `is_a` target type fixed structurally by id-prefix |
| Property chains (OWL 2) | ✓ | ✗ | ✗ | ✗ | ✓ — opt-in `chain` declarations on PredicateAtoms (e.g. `part_of := component_of ∘ component_of+`); asserted-edge matching, never composition-kind output |
| Class expressions (intersection / union / complement) | ✓ | ✗ | ✗ | ✗ | ✗ — deliberate; over-engineering for contestable knowledge |
| Reasoner / automated entailment | full FOL (decidable fragment) | partial | ✗ | ✗ | ✓ — fixed three-tier rule set (T1/T2/T3) required by spec; no general theorem prover |
| Open-world assumption | default | ✓ | ✗ | closed | scoped closed world; `hanging` is explicit "don't know" |
| Unique-name assumption | optional | optional | n/a | usually | no — aliases imply non-UNA |
| Per-claim provenance | bolt-on (named graphs) | bolt-on | ✗ | partial | ✓ native — `sources`, `confidence` |
| Operation-level provenance / change correlation | ✗ | ✗ | ✗ | ✗ | ✓ — `correlation_id` (`ingest:` / `request:` / `system:`) on every change event and on `status`; full operation trace |
| Lifecycle states | ✗ | ✗ | partial (`skos:status`) | ✗ | ✓ native — active / hanging / deferred / deprecated + removal outcomes |
| Time scope / temporal validity | workarounds | workarounds | ✗ | partial | ✓ — `Scope.time_scope` |
| Snapshot isolation / coherent versioning | ✗ | ✗ | ✗ | ✗ | ✓ — working→canonical publish; per-operation serialization; backend-neutral |
| Authoring workflow | ✗ | ✗ | ✗ | ✗ | ✓ — atoms-as-SoR + snapshot publish (git-PR in git-backed deployments) |
| Conflict pipeline as consistency check | reasoner-driven | ✗ | ✗ | ✗ | ✓ — structured outcome family (9 stages, ~22 outcomes, 3 resolution modes) |
| Tree-bounded parallel ontologies | ✗ | ✗ | partial (concept schemes) | ✗ | ✓ — explicit `Scope.tree`; same-claim collapses within, `kind=equivalence` connects across |
| Primary / secondary classification | ✗ — `subClassOf` is uniformly multi-valued | ✗ | partial | ✗ | ✓ — single `is_a` + multi via `kind=specialization` |
| Identity + Conflict test for atom-vs-attribute | ✗ | ✗ | ✗ | ✗ | ✓ — formal principle |
| Derived atoms as first-class | ✗ — inferred axioms are reasoner output, not model citizens | ✗ | ✗ | ✗ | ✓ — `derivation` struct, hidden by default in the projection facet |

## Where OWL is strictly more rigorous

- **Universal restrictions** (∀P.C / allValuesFrom — "every value of P is a C"). Not modeled; we have existential and qualified-cardinality restrictions but not all-values-from.
- **Class expressions.** No intersection, union, complement, or other algebraic constructions over classes.
- **Full FOL reasoning.** We have a fixed-rule three-tier entailment engine; OWL DL has a decidable FOL reasoner that can prove arbitrary logical consequences within its fragment.

These are deliberate omissions for the contestable-organizational-knowledge use case. They primarily benefit formal-logic applications (medical ontologies, regulatory rules) — not the domain we target.

## Where we are strictly more rigorous than OWL

- **Per-claim provenance.** Every atom carries `sources` and `confidence`. OWL treats provenance as out-of-band (named graphs, PROV-O annotations).
- **Operation-level correlation / trace.** Every change event and every atom's `status` carries a `correlation_id`, so the full cross-container effect of one ingest, request, or internal job is reconstructable by filter. OWL has no notion of change operations at all.
- **Lifecycle states.** active / hanging / deferred / deprecated + removal outcomes are first-class, each transition tagged with the operation that caused it. OWL has no notion of a "tentative" or "questioned" axiom.
- **Time scope.** `Scope.time_scope` makes temporal validity native. OWL needs workarounds (4D-ism, temporal annotations).
- **Snapshot isolation + authoring workflow.** Atoms-as-SoR with working→canonical publish (per-operation serialization, atomic commit) is the canonical change path, backend-neutral. OWL is silent on how changes are made or isolated.
- **Tree-bounded ontologies.** Parallel discourses with same-claim collapse inside and `kind=equivalence` across. OWL has no equivalent of "these two ontologies describe the same world from different angles — don't auto-merge."
- **Primary + secondary classification distinction.** Single `is_a` (rdf:type-like) plus multi-classification via `kind=specialization` (rdfs:subClassOf-like). OWL's `subClassOf` is uniformly multi-valued — communities re-invent the "primary class" distinction by convention.
- **Structured conflict outcome family.** A pipeline of named outcomes with documented resolution paths and three resolution modes (auto-resolve / soft-prompt / hard-block) — see [06](../../spec/06-conflict-classification.md). OWL's reasoner returns "inconsistent" — one bit; ours returns actionable categories.
- **Identity + Conflict test.** A formal principle for deciding when to atomize a value vs. keep it as an attribute. OWL's datatype-vs-object-property choice is a typing decision, not a methodology.
- **Derived atoms as first-class citizens.** Entailed atoms are part of the model with `derivation` provenance, distinguishable from asserted ones. OWL inferred axioms exist only as reasoner output.

## Where we now match OWL

- **Classes / types** — full coverage via `AtomType` + ClassificationAtom hierarchy.
- **Subclass / generalization** — full via `is_a` + `kind=specialization`, with auto-derived transitive closure.
- **Subproperty hierarchy** — full via sub-predicate chains (`predicate.is_a → predicate`); predicate *kinds* stay a closed enum of six by design.
- **Properties as first-class** — full via `RelationAtom`.
- **Property characteristics** — 6 of 7 (only reflexive skipped, deliberately): functional, inverse_functional, transitive, symmetric, asymmetric, irreflexive.
- **Inverse pairs** — full via `inverse_of` with auto-materialization (T1).
- **Equivalence** — richer than OWL: three distinct mechanisms operating at different levels (names, atoms, ingestion).
- **Disjointness** — full via `disjoint_with` on ClassificationSchema, plus propagation through `is_a` chains (T3).
- **Property chains** — opt-in `chain` declarations on PredicateAtoms (`spec/03 § Chain derivation`); the standard library ships `part_of` / `has_part` chained over `component_of`. Narrower than OWL 2's by design: chains bind concrete predicates (never kinds), match asserted edges only, and can never output a composition-family edge — the combination OWL 2 DL forbids (transitive + functional) is structurally unrepresentable.
- **Existential + qualified cardinality restrictions** — via `required_relations` / `optional_relations` (`min`/`max` to a `target_classification`). Covers someValuesFrom and qualified min/max/exact; only allValuesFrom (universal) is absent.
- **Domain / range constraints** — full on predicates (`domain_classifications` / `range_classifications`) and on referential classifications (subject reference).
- **Automated entailment** — narrow but rigorous: T1+T2+T3 are spec-required rules, not a general theorem prover.

## Where we differ deliberately

- **Open-world vs. scoped closed world.** OWL OWA confuses reviewers who think in concrete-snapshot terms. We use "claim is present in the snapshot or it isn't" + `hanging` as the explicit "don't know."
- **Reasoner-driven vs. human-in-loop resolution.** OWL: the reasoner finds inconsistencies; the modeler fixes the ontology. Ours: the pipeline surfaces structured outcomes; humans resolve via the snapshot/PR workflow. Better fit for contestable knowledge.
- **No class algebra, no universal restrictions.** Natural-language equivalents are expressed as additional claims rather than algebraic class definitions.

## Comparison summary

| Dimension | Verdict |
|---|---|
| **Rigor on classical ontology** (classes, sub-class and sub-predicate hierarchies, properties, characteristics, property chains, equivalence, disjointness, existential + qualified cardinality, domain/range, entailment) | At parity with OWL DL on most features; strictly weaker on universal restrictions, class expressions, and full FOL reasoning. |
| **Rigor on knowledge-workflow features** (provenance, operation correlation, lifecycle, time, snapshot versioning, authoring, parallel discourses, structured conflict) | Strictly more rigorous than OWL/RDFS/SKOS/PG — these features are absent or bolt-on there. |
| **Capabilities exhaustiveness** | Covers the large majority of typical knowledge-modeling needs. Missing the deep-logic features (universal restrictions, class algebra, full FOL). |
| **Positioning** | SKOS + property graph + first-class workflow + selective OWL features (closed enum of predicate kinds, the six property characteristics, qualified existential/cardinality restrictions, opt-in property chains, atom-level entailment) + tree partitioning. Not OWL DL — and not trying to be. |

## Where to evolve next

Worth considering for future versions:
- **Universal restrictions** (∀P.C / allValuesFrom) — currently only existential and qualified-cardinality restrictions are modeled. Under review alongside a possible explicit class-expression syntax for `∃P.C` / `∀P.C` / `(=n P)`.
- **Per-kind domain/range defaults** at the predicate-kind level (predicates already carry `domain_classifications` / `range_classifications`; defaulting per kind is open).
- **Blanket self-transitivity audit** — with explicit chains as the opt-in closure mechanism, revisit which predicate kinds should keep blanket transitivity (is `member_of` truly self-transitive?).

Already present (no longer future work): sub-predicate hierarchy, edge-level qualified cardinality (`required_relations` min/max), existential restrictions, and property chains (`chain` declarations, e.g. "is-deployed-in" = "is-implemented-by ∘ is-deployed-via").

Reasoner-grade FOL is *not* on the roadmap — the human-resolution stance is a deliberate design choice, not a limitation.
