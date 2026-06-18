# aala

**A knowledge compiler — it turns the decisions scattered across your chat, docs, and reviews into a canonical, queryable system of record.**

[![Diagrams](https://github.com/Gubbi/aala/actions/workflows/likec4-diagrams.yml/badge.svg)](https://github.com/Gubbi/aala/actions/workflows/likec4-diagrams.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

> **Status:** `aala-v0.1` — a pre-publish draft specification, under active refinement. Expect breaking changes.

## The Problem

Organizations decide faster than they can write things down. The *why* behind an architecture, the constraint someone raised in a meeting, the dependency that made a choice load-bearing — these live in Slack threads, transcripts, and PR comments, and they decay the moment the conversation scrolls away. Wikis go stale because prose has to be maintained by hand. Search returns documents, not answers. And when a decision changes, nobody can see everything it quietly invalidated.

The knowledge exists. It's just never *compiled* into something you can query, trust, and keep current.

## What aala Does

aala treats every incoming fragment — a chat thread, a transcript, a PR discussion — as evidence, and extracts the **claims** inside it as small, typed, individually-addressable facts called **atoms**. Atoms are the system of record. Everything else is derived from them:

```
  chat threads ─┐
  transcripts  ─┤      ┌─────────────┐      atoms ── the system of record
  PR reviews   ─┼────► │    aala     │ ───► (typed claims + relationships)
  design docs  ─┤      │  compiler   │             │
  ad-hoc notes ─┘      └─────────────┘             ├─► prose  (review · LLM grounding · export)
                                                   ├─► navigation & search
                                                   └─► blast-radius impact analysis
```

Because prose is **projected** from atoms on demand rather than authored, the documents can never drift from what's actually known. A conflict pipeline checks every new claim against what already exists — flagging duplicates, refinements, and contradictions. And when a decision changes, blast-radius analysis shows exactly which other claims are now in question.

## What Makes aala Different

- **Atoms, not documents, are the source of truth.** Most systems store prose and bolt search on top; aala stores typed claims and generates prose from them — so the readable layer is always faithful to the underlying facts.
- **A closed, typed grammar — not freeform tags.** Five atom types, ten relationship kinds, an exhaustive conflict taxonomy. Knowledge is structured enough to validate, reason over, and detect contradictions in.
- **Knowledge is reviewed like code.** Changes land in working snapshots, surface as diffs, and publish by fast-forward — the pull-request workflow, applied to what your organization knows.
- **A specification, not a product.** aala is an implementation-independent contract. Any vendor can build a conforming engine, and conforming engines interoperate.

## Who It's For

- **Teams** losing institutional memory to chat scrollback and stale wikis.
- **Agent builders** who need a grounded, provenance-backed knowledge source instead of a hallucination-prone document pile.
- **Implementers** building a knowledge engine to the spec — or comparing one against it.

## A Concrete Example

A team decides in Slack: *"Let's switch the config format from JSON to YAML."*

1. **Ingest.** The thread enters aala as a fragment, preserved verbatim for provenance.
2. **Extract.** A claim atom is produced: *config format → YAML*, classified as a `Decision`, linked back to the message that asserted it.
3. **Conflict.** The pipeline finds the existing *config format → JSON* decision and classifies the newcomer as a **contradiction**, not a duplicate — so nothing is silently overwritten.
4. **Review.** In a working snapshot, a reviewer confirms the switch. The JSON decision transitions to `deprecated`; the YAML decision becomes active.
5. **Blast radius.** Deprecating the old decision cascades: the *"chose this parser because the format is JSON"* atom now stands on a dead premise, so it surfaces as needing review — automatically.
6. **Project & publish.** The affected docs regenerate from the new atom set, the diff is reviewed, and publishing fast-forwards it into the canonical record.

No one hand-edited a wiki. The decision, its contradiction, its downstream impact, and the refreshed prose all fell out of compiling one Slack thread.

## The Specification

`spec/` and `interfaces/` together are the complete, self-contained contract — an implementer needs nothing else.

| Path | What it is |
|---|---|
| [`spec/`](./spec/) | The normative specification — data model, snapshots, delta streams, conflict classification, atom lifecycle, blast radius, error model, conformance, wire profiles, and the registry appendices. |
| [`interfaces/`](./interfaces/) | The binding interface contracts — one file per container (signatures, errors, idempotency, concurrency, events). |
| [`docs/analysis/`](./docs/analysis/) | Informative analyses — e.g. the [ontology comparison](./docs/analysis/ontology-comparison.md) (vs OWL/RDFS/SKOS/LPGs) and the [external-agent integration pattern](./docs/analysis/agent-integration-pattern.md). |
| [`docs/`](./docs/) | Supporting material and design lineage — the earlier C4 design (`L1`/`L2`/`L3`) and the LikeC4 sources ([`docs/likec4/`](./docs/likec4/)) that seeded the spec. Lineage only; not normative. |

**Start here:** [`spec/00-introduction.md`](./spec/00-introduction.md) (scope) → [`spec/03-data-model.md`](./spec/03-data-model.md) (atoms, classifications, the ten predicate kinds) → [`spec/02-conformance.md`](./spec/02-conformance.md) (what "conformant" means) → [`interfaces/`](./interfaces/) (the contracts). The closed registries live in one place: [Appendix A](./spec/appendix-a-registries.md); the default projection format is [Appendix B — OKF](./spec/appendix-b-okf-profile.md).

## Project Status

`aala-v0.1`, pre-publish. The specification is substantially complete and internally consistent, but it has not been ratified, and breaking changes are expected as it's refined and validated against real implementations.

## Contributing

Issues and discussion are welcome — particularly on ambiguities, gaps, and conformance edge cases. This is a specification, so the highest-value contributions are careful reads that find where it's underspecified or self-contradictory.

## License

Released under the [MIT License](./LICENSE).

## Learn More

- [Specification index](./spec/README.md)
- [Interface contracts index](./interfaces/README.md)
- [How external agents consume aala](./docs/analysis/agent-integration-pattern.md)
- [aala vs. OWL / RDFS / SKOS / property graphs](./docs/analysis/ontology-comparison.md)
