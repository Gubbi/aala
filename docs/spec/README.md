# aala — Specification

This folder is the **binding specification** for aala. It defines what a conformant implementation must do.

## Audience

| Role | Reads this for |
|---|---|
| **Implementer** | What they're building. The chapters here, plus [`docs/interfaces/`](../interfaces/), define the contract that callers will hold the implementation to. |
| **Deployer** | What they can expect from any conformant implementation, and what they must configure (LLM gateway, snapshot backing, wire profiles, etc.). |
| **Spec reviewer** | Where to ground a critique or proposal. |
| **Tool / client author** | What aala will and won't do under any conformant implementation. |

## How to read this

Start with [`00-introduction.md`](./00-introduction.md) → [`01-conventions.md`](./01-conventions.md) → [`02-conformance.md`](./02-conformance.md). Those three frame everything else. Then read by topic.

| # | Chapter | Topic |
|---|---|---|
| 00 | [Introduction](./00-introduction.md) | What aala is, normative vs informative scope, how the spec relates to the interfaces |
| 01 | [Conventions](./01-conventions.md) | Normative keywords (BCP 14), terminology, notation |
| 02 | [Conformance](./02-conformance.md) | What makes an implementation conformant; capability + wire-profile declarations; the access-control model |
| 03 | [Data Model](./03-data-model.md) | Atom types (entity / relation / classification / predicate / predicate_kind), schema model, statuses, removal outcomes, standard library |
| 04 | [Snapshots](./04-snapshots.md) | Snapshot lifecycle and coherence invariants |
| 05 | [Delta Streams](./05-delta-streams.md) | `changes_since(ref)` ordering, replay equivalence |
| 06 | [Conflict Classification](./06-conflict-classification.md) | Conflict outcomes (asserted + entailment-derived), resolution modes |
| 07 | [Atom Lifecycle](./07-atom-lifecycle.md) | State machine, transitions, cascade rules per predicate kind |
| 08 | [Blast Radius](./08-blast-radius.md) | Impact analysis, pipeline stages, iteration |
| 09 | [LLM Gateway](./09-llm-gateway.md) | Standard use-case keys, schema validation, observability |
| 10 | [Error Model](./10-error-model.md) | Three-tier error chain, classification, mandated OTel emission, idempotency |
| 11 | [Versioning](./11-versioning.md) | Backward-compatible vs breaking, version negotiation |
| 12 | [Edge Cases](./12-edge-cases.md) | Concurrent operations, malformed inputs, convergence, snapshot transitions |
| 13 | [Wire Profiles](./13-wire-profiles.md) | Serialization + transport bindings for cross-implementation interop |
| A | [Appendix A — Registries](./appendix-a-registries.md) | The spec's closed registries — single-home enumerations the chapters reference |
| B | [Appendix B — OKF Profile](./appendix-b-okf-profile.md) | The default projection output profile — the OKF interchange rendering, versioned independently |

## Relationship to the interfaces

- [`docs/interfaces/`](../interfaces/) — normative. Each container's method signatures, error declarations, idempotency, and concurrency rules are binding.
- [`docs/analysis/`](../analysis/) — informative; forward-facing analyses.

The spec and the interface files together are the complete, self-contained contract. The spec adds the cross-cutting invariants and rules the interfaces leave implicit — atom data model, classification semantics, lifecycle, ordering, conformance levels, error retry, versioning, edge cases, wire profiles. Ownership and precedence between the two are defined in [01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership); the container partition is defined in [02 — Conformance](./02-conformance.md).
