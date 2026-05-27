# 00 — Introduction

## What aala is

aala is a system that turns fragmented organizational inputs (chat threads, meeting transcripts, version-control discussions, scattered docs) into a canonical, queryable knowledge layer made of structured **atoms** (claims). Atoms are the System of Record; prose **projections** are derived views; **synthesis** composes responses on demand. Multi-axis **navigation** and **blast-radius** impact analysis surface what's affected when a decision changes.

This specification defines the contract that any conformant implementation of aala must honor.

## What's in scope

This spec binds at the level of:

- The **container set**: which capability containers MUST or MAY be present (see [02 — Conformance](./02-conformance.md)).
- The **interface contracts**: each container's methods, signatures, error variants, idempotency, and concurrency rules. The binding signatures live in [`docs/interfaces/`](../interfaces/) and are referenced from spec chapters.
- The **data model**: atom shape, status, removal outcomes, supporting types.
- The **behavioral invariants**: snapshot coherence, delta-stream ordering, replay equivalence, classification semantics, atom lifecycle, error model.
- The **published registries**: the standard LLM Gateway use-case keys, the atom type taxonomy.

## What's out of scope

The spec does not bind:

- **Internal component layouts** within a container. `docs/L3/` describes one valid internal organization; implementations are free to structure differently as long as the container-boundary contracts are honored.
- **Wire formats**. The conceptual data model is normative; the on-the-wire serialization (YAML, JSON, protobuf, binary) is an implementation choice.
- **Specific tools** (LiteLLM, git, particular databases, particular LLM providers). Constrained only by what they realize from the container interfaces.
- **Deployment topology**. Single-process, multi-process, distributed — all permitted as long as the contracts hold.
- **Storage backends**. As long as the snapshot lifecycle and delta-stream semantics hold.

## Relationship to design documents

The spec is the binding contract. The design documents serve different purposes:

| Document | Status | Use |
|---|---|---|
| [`docs/L1-context.md`](../L1-context.md) | Informative | System-level framing, personas, scope |
| [`docs/L2/`](../L2/) | Mixed: container partition + interface boundaries are **normative**; narrative + variation points are informative | Architectural understanding |
| [`docs/L3/`](../L3/) | **Informative only** | One example internal component organization. Useful as a reference; not binding. |
| [`docs/interfaces/`](../interfaces/) | **Normative** | The binding signatures and contracts. Spec chapters reference these. |
| `docs/spec/` (this folder) | **Normative** | The contract this implementation must honor. |

## How to use this spec

- An **implementer** building aala MUST honor every "MUST" in this spec and SHOULD honor every "SHOULD". MAY clauses are permitted but not required.
- A **client / caller** can rely on every "MUST" being true in any conformant implementation.
- A **deployer** can compare implementations by their **conformance declaration** (see [02 — Conformance](./02-conformance.md)) — which capabilities are wired and at what depth.

The next chapter ([01 — Conventions](./01-conventions.md)) defines the keywords and terminology this spec uses.
