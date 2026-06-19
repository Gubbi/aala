# Scope Boundaries — What's Not in L2

Concerns that are intentionally not part of the L2 architecture, with brief reasoning. Each is either deferred to a future iteration, owned by an implementation, or out of aala's responsibility entirely.

## Deferred — to be designed properly later

| Concern | Note |
|---|---|
| **Code as a source of truth** | Atoms cover planning-level knowledge (decisions, constraints, definitions, behaviors, capabilities, scenarios). Implementation code is its own SoR. Bridging the two — e.g., resolving a planning-tree decision's premise against the implementation-tree atoms extracted from code — is future work. The tree-partitioning + `kind=equivalence` model gives us the substrate to express the bridge; the extraction tooling for code-derived atoms is not yet built. |
| **Cross-deployment federation** | An aala deployment manages one logical organization's atoms. Cross-deployment queries, shared glossaries across organizations, and inter-deployment provenance are future. |
| **Class algebra and universal restrictions** | Classification compositions in the current model support existential restrictions and min/max cardinality on `required_relations`. Class algebra (intersection, union, complement) and universal restrictions (`∀P.C`) are deliberately out of v0.1 scope per the FOL-compatibility analysis in [`analysis/ontology-comparison.md`](../../analysis/ontology-comparison.md). |
| **Property chains** | OWL 2's property chain axioms (`hasUncle = hasParent ∘ hasBrother`) are not modeled in v0.1. Predicate compositions are anticipated as a future extension on the predicate plane. |
| **Cross-cutting concerns at proper depth** | [Multi-tenancy](./12-cross-cutting.md#multi-tenancy), [privacy boundaries](./12-cross-cutting.md#privacy-boundaries), [authentication / authorization](./12-cross-cutting.md#authentication-and-authorization), and [observability / tracing](./12-cross-cutting.md#observability-and-tracing) are sketched in `12-cross-cutting.md` but need dedicated design documents. |

## Owned by implementations, not L2

| Concern | Owned by |
|---|---|
| Concrete persistence backend for snapshots | The implementation's choice (git working tree, versioning DB, content-addressed manifest, etc.). |
| Concrete LLM gateway tool | The implementer (LiteLLM / OpenRouter / Portkey / custom). |
| Entailment caching strategy | The implementation — lazy, eager, or hybrid materialization of derived atoms. Spec mandates the entailment rules; caching is implementation. |
| External API protocol | The implementation (MCP tools, HTTP, CLI, embedded library). |
| Build, packaging, install, update mechanics | The implementation's deployment story (e.g., brew install for a single-user MCP server; container images for a SaaS). |
| Client SDKs | Whoever ships a client; aala publishes an interface, not SDKs. |

## Not aala's responsibility

| Concern | Why |
|---|---|
| **Authoritative truth resolution** | The Conflict pipeline (inside Atoms) surfaces structured outcomes with resolution modes (auto-resolve / soft-prompt / hard-block) and documented resolution paths. Humans decide truth for soft-prompts; structural hard-blocks are rejected at proposal time. aala makes the conflict cheap to notice and review; it does not silently pick a side. |
| **General-purpose workflow / DAG orchestration** | Orchestration coordinates aala's own lifecycle (snapshots, trees, ingest, query). Long-running multi-step business workflows beyond ingest/query are out of scope; integrate with an external workflow engine if needed. |
| **Real-time collaborative editing of atoms or projections** | Reviews happen via snapshot diffs. Live multiplayer editing of the canonical record is out of scope; agent-mediated edits via the normal `Atoms.propose` flow is the supported pattern. |
| **Replacing the deployer's existing review tools** | A deployer can review via `git diff`, their editor's diff view, the agent presenting summaries, or any tool that reads the snapshot. aala does not ship its own diff UI. |
| **Deciding what's a "good" answer** | [Quality](./10-quality.md) measures against benchmarks and golden sets that the implementer and deployer author. aala does not have an opinion absent those configurations. |
| **Full FOL reasoning** | The entailment engine computes a fixed, decidable rule set (T1+T2+T3). A general theorem prover is explicitly out of scope. Structural conflict pipeline + human review replace reasoner-driven verdicts for contestable knowledge. |
