# aala — Knowledge Compiler

**aala** turns fragmented organizational inputs — chat threads, meeting transcripts, version-control discussions, scattered docs — into a canonical, queryable **knowledge layer** made of structured **atoms** (typed claims).

Atoms are the **System of Record**. Prose is never the source of truth: it is a **projection** derived on demand from atoms, for human review, LLM grounding, and export/interchange. Change flows through snapshot-and-review (git-PR-style) workflows; a conflict pipeline classifies every incoming claim against what's already known; and blast-radius analysis surfaces what's affected when a decision changes.

This repository is a **specification** — a self-contained, implementation-independent contract, in the spirit of the Iceberg and DuckDB specs. It is the deliverable; conforming implementations follow.

> **Status:** `aala-v0.1` — pre-publish draft, under active refinement.

## What's here

| Path | What it is |
|---|---|
| [`spec/`](./spec/) | **The normative specification** — data model, snapshots, delta streams, conflict classification, atom lifecycle, blast radius, error model, conformance, wire profiles, and the registry appendices. Start at [`spec/README.md`](./spec/README.md). |
| [`interfaces/`](./interfaces/) | **The binding interface contracts** — one file per container (method signatures, per-method errors, idempotency, concurrency, event unions). Start at [`interfaces/README.md`](./interfaces/README.md). |
| [`docs/analysis/`](./docs/analysis/) | Informative analyses — e.g. [ontology comparison](./docs/analysis/ontology-comparison.md) (vs OWL/RDFS/SKOS/LPGs) and the [external-agent integration pattern](./docs/analysis/agent-integration-pattern.md). |
| [`docs/`](./docs/) | Supporting material and **design lineage** — the earlier C4 design (`L1`/`L2`/`L3`) and LikeC4 sources ([`likec4/`](./likec4/)) that seeded the spec. Lineage only; not normative, and may have drifted. |

The specification is complete on its own: **`spec/` + `interfaces/` are everything an implementer needs.**

## Reading guide

1. [`spec/00-introduction.md`](./spec/00-introduction.md) — what aala is, what's in and out of scope.
2. [`spec/01-conventions.md`](./spec/01-conventions.md) — normative keywords and the one-home ownership rule.
3. [`spec/03-data-model.md`](./spec/03-data-model.md) — the five atom types, classifications, the ten predicate kinds, entailment.
4. [`spec/02-conformance.md`](./spec/02-conformance.md) — required containers, the Projection facet, access control, what "conformant" means.
5. [`interfaces/`](./interfaces/) — the contracts, container by container.

The closed registries (predicate kinds, conflict outcomes, error activities and end-states, the standard library, identifiers, use-case keys) live in one place: [Appendix A — Registries](./spec/appendix-a-registries.md). The default projection output profile is defined in [Appendix B — OKF Profile](./spec/appendix-b-okf-profile.md).

## Core ideas

- **Atoms are the System of Record.** Five concrete types — `entity`, `relation`, `classification`, `predicate`, `predicate_kind` — under one typed grammar. Prose is projected from them, never the other way around.
- **Closed, registered vocabularies.** Ten predicate kinds (in inverse pairs), an exhaustive conflict-outcome set, a fixed error-activity vocabulary — each with a single normative home and conformance lockstep.
- **Snapshot-and-review.** Working snapshots fork from the canonical one; changes are reviewed as diffs and published by fast-forward — the familiar pull-request shape, applied to knowledge.
- **Derived read models.** Projection, navigation, and blast-radius are change-log-driven materialized views over the atom store, not hand-authored artifacts.
- **A three-tier error model and mandated OpenTelemetry** give independently built, composable containers a coherent contract and a uniform observability baseline.

---

*License: not yet specified.*
