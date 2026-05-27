# aala — Specification

This folder is the **binding specification** for aala. It defines what a conformant implementation must do.

## Audience

| Role | Reads this for |
|---|---|
| **Implementer** | What they're building. The chapters here, plus [`docs/interfaces/`](../interfaces/), define the contract that callers will hold the implementation to. |
| **Deployer** | What they can expect from any conformant implementation, and what they must configure (LLM gateway, snapshot backing, etc.). |
| **Spec reviewer** | Where to ground a critique or proposal. |
| **Tool / client author** | What aala will and won't do under any conformant implementation. |

## How to read this

Start with [`00-introduction.md`](./00-introduction.md) → [`01-conventions.md`](./01-conventions.md) → [`02-conformance.md`](./02-conformance.md). Those three frame everything else. Then read by topic.

| # | Chapter | Topic |
|---|---|---|
| 00 | [Introduction](./00-introduction.md) | What aala is, normative vs informative scope, how the spec relates to L2 / L3 / interfaces |
| 01 | [Conventions](./01-conventions.md) | RFC 2119 keywords, terminology, notation |
| 02 | [Conformance](./02-conformance.md) | What makes an implementation conformant; capability declarations |
| 03 | [Data Model](./03-data-model.md) | Atom shape, types, statuses, removal outcomes |
| 04 | [Snapshots](./04-snapshots.md) | Snapshot lifecycle and coherence invariants |
| 05 | [Delta Streams](./05-delta-streams.md) | `changes_since(ref)` ordering, replay equivalence |
| 06 | [Conflict Classification](./06-conflict-classification.md) | Exact rules for the 8 conflict outcomes |
| 07 | [Atom Lifecycle](./07-atom-lifecycle.md) | State machine, transitions, cascade rules |
| 08 | [Blast Radius](./08-blast-radius.md) | Impact analysis, pipeline stages, iteration |
| 09 | [LLM Gateway](./09-llm-gateway.md) | 9 standard use-case keys, schema validation, observability |
| 10 | [Error Model](./10-error-model.md) | Error variants, retry semantics, idempotency |
| 11 | [Versioning](./11-versioning.md) | Patch-compatible vs breaking, version negotiation |
| 12 | [Edge Cases](./12-edge-cases.md) | Concurrent operations, malformed inputs, convergence, snapshot transitions |

## Relationship to the design docs

- [`docs/L1-context.md`](../L1-context.md) — informative; system-level framing.
- [`docs/L2/`](../L2/) — informative for narrative, **normative for container partitioning and interface boundaries**. The 9 containers and which are optional / non-optional are part of the spec.
- [`docs/L3/`](../L3/) — informative only. Describes one valid internal component organization. Implementations are free to structure differently.
- [`docs/interfaces/`](../interfaces/) — normative. Each container's method signatures, error variants, idempotency, and concurrency rules are binding.

The spec adds the cross-cutting invariants and rules the interfaces leave implicit (classification semantics, lifecycle, ordering, conformance levels, error retry, versioning, edge cases).
