# 00 — Introduction

## What aala is

aala is a system that turns fragmented organizational inputs (chat threads, meeting transcripts, version-control discussions, scattered docs) into a canonical, queryable knowledge layer made of structured **atoms**. Atoms are the System of Record; prose **projections** are derived read views each contentful container exposes over its own content — for human review, LLM grounding, and **export/interchange** (the projection materializes under a single configured output profile, an interchange format by default — see [02 — Conformance § Required facet — Projection](./02-conformance.md#required-facet--projection)). Multi-axis **navigation** and **blast-radius** impact analysis surface what's affected when a decision changes. Composing answers or documents from these read surfaces is the job of external agents consuming aala (see [`analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md)).

The atom model is a typed grammar: five concrete atom types (`entity`, `relation`, `classification`, `predicate`, `predicate_kind`) compose into a graph where classifications carry structural schemas and predicates carry typed relationships. The model is FOL-friendly in its substrate while keeping resolution human-in-loop for contestable knowledge.

This specification defines the contract that any conformant implementation of aala must honor.

## What's in scope

This spec binds at the level of:

- The **container set** — which capability containers MUST or MAY be present (see [02 — Conformance](./02-conformance.md)).
- The **interface contracts** — each container's methods, signatures, `(activity, end_state)` error declarations, idempotency, and concurrency rules. The binding signatures live in [`interfaces/`](../interfaces/) and are referenced from spec chapters.
- The **data model** — the five concrete atom types, the schema model, classification semantics, predicate kinds, status states and removal outcomes, the entailment tiers.
- The **behavioral invariants** — snapshot coherence, Delta-Stream ordering, replay equivalence, atom lifecycle, conflict classification and resolution modes, blast radius, error model.
- The **published registries** — the closed enumerations consolidated in [Appendix A — Registries](./appendix-a-registries.md): the containers, identifier minting, the predicate kinds, the standard library every tree initializes with, the conflict outcomes, the LLM Gateway use-case keys, and the error-model vocabularies (activities, operations, end-states).
- The **access-control model** — session-carried grants of `(snapshot, operation type)` pairs, checked at the perimeter on every call; each interface method declares its operation type. See [02 — Conformance § Access control](./02-conformance.md#access-control).
- The **wire profiles** — at least one wire serialization + transport binding for cross-implementation interop. See [13 — Wire Profiles](./13-wire-profiles.md).

## What's out of scope

The spec does not bind:

- **Internal component layouts** within a container. Implementations are free to structure container internals however they choose, as long as the container-boundary contracts are honored.
- **Choice of wire profile beyond requiring at least one `rpc`-class profile**. Implementations declare which wire profiles they support in `Orchestration.capabilities()`; the spec defines the profiles and their classes ([13 — Wire Profiles](./13-wire-profiles.md#profile-classes)) but doesn't mandate which one(s) an implementation chooses.
- **Specific tools** (particular databases, particular LLM providers, particular vector indices). Constrained only by what they realize from the container interfaces.
- **Deployment topology**. Single-process, multi-process, distributed — all permitted as long as the contracts hold.
- **Storage backends**. As long as the snapshot lifecycle and Delta-Stream semantics hold.
- **Caching strategy for derived (entailed) atoms**. The spec mandates which entailments must be computed; implementations choose lazy, eager, or hybrid materialization.
- **Identity and access policy.** Caller identity, token formats, identity providers, and the role/policy machinery that *produces* a session's grants. The spec binds the grant check itself ([02 — Conformance § Access control](./02-conformance.md#access-control)) and, per wire profile, how establishment context is conveyed ([13 — Wire Profiles](./13-wire-profiles.md)); how a deployment decides who receives which grants is its own concern.
- **Multi-tenancy.** This contract describes **one logical deployment** — a single tenant's knowledge layer, with its own snapshot space, trees, standard library, and canonical pointer. A multi-tenant offering (e.g., a SaaS) operates N logical deployments behind its own tenancy layer: that layer attaches tenant identity and routes each caller to its deployment at session establishment ([04 — Snapshots § Session binding](./04-snapshots.md#session-binding)), and none of it is bound here. No shape in this spec or in [`interfaces/`](../interfaces/) carries a tenant identifier. The conformance consequence is stated in [02 — Conformance § Single-deployment scope](./02-conformance.md#single-deployment-scope).
- **The session-context payload.** The grants a session carries, the caller's identity, the active tenant, and any other deployment-supplied context reach the containers through a **session-context payload** passed alongside each call. It is **schema-free** and **signed by the deployment layer** — the signature attests that the wrapping deployment / session gateway produced it — and its contents are an **implementation-specific** agreement between that layer and the containers, **not part of the external interface contract**. The spec binds only what containers do with the parts it names — chiefly the grant check ([02 — Conformance § Access control](./02-conformance.md#access-control)); the payload's concrete shape, its additional fields (tenant, identity, request metadata), and its signing scheme are the implementation's choice.
- **Historical reconstruction.** The API surface addresses the current state of the snapshot it runs against. Atoms are mutable ([03 — Data Model § Mutability and revision](./03-data-model.md#mutability-and-revision)); previously published snapshots carry what was. Querying past states is a store concern, not an API concern — a deployment whose persistence keeps history (e.g., one built on the git-backed wire profile) recovers it from the store — and the Delta Stream is forward catch-up, not a historical query surface ([05 — Delta Streams](./05-delta-streams.md)).

## Relationship to other documents

The specification is self-contained: `spec/` and `interfaces/` together are the complete binding contract. An implementer needs nothing else.

| Document | Status | Use |
|---|---|---|
| `spec/` (this folder) | **Normative** | Semantics, invariants, registries, lifecycles, conformance. |
| [`interfaces/`](../interfaces/) | **Normative** | The binding signatures and per-method contracts. Spec chapters reference these. |
| [`analysis/`](../analysis/) | Informative | Forward-facing analyses (e.g., ontology comparison). Not binding. |

Nothing outside these folders carries normative weight. The ownership split and precedence rule between spec chapters and interface files are defined in [01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership). The container partition itself — which containers exist and which are optional — is defined in [02 — Conformance](./02-conformance.md).

## Who reads what

- An **implementer** building aala produces software that satisfies every normative requirement in this spec. The normative keywords (BCP 14 — RFC 2119 + RFC 8174) are declared once, in [01 — Conventions](./01-conventions.md).
- A **client / caller** writing against aala can rely on every MUST clause being true in any conformant implementation and can defensively reason about SHOULD / MAY by consulting [`Orchestration.capabilities()`](../interfaces/orchestration.md#capabilities).
- A **deployer** running an aala instance compares implementations by their conformance declaration (see [02 — Conformance](./02-conformance.md)) — which containers are wired, what wire profiles are supported, what LLM gateway tool is configured.

The next chapter ([01 — Conventions](./01-conventions.md)) defines the keywords and terminology this spec uses.
