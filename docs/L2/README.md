# L2 — Container Architecture

**Knowledge Compiler — conceptual container architecture.**

This folder documents the *superset* of containers a full aala deployment may include. Specific deployable shapes (local-MCP+agent, SaaS multi-tenant, real-time meeting capture, etc.) realize subsets of these containers — see `docs/implementations/<impl>/` for those.

The container list below is a consequence of applying the principles in [`01-principles.md`](./01-principles.md), not an arbitrary partition. Read that first if you're new to this design.

## Container list

| # | Container | Optional? | Concern |
|---|---|---|---|
| 1 | [Ingestion](./02-ingestion.md) | non-optional | Bring raw inputs into the system; normalize; persist. |
| 2 | [Atoms](./03-atoms.md) | non-optional | Canonical record of claims. Owns the atom schema, CRUD, Extraction (C), Conflict detection (status / supersession), alias resolution, reference traversal. |
| 3 | [Projection](./04-projection.md) | non-optional | Render atoms into human-readable prose (markdown). Owns prose files + reverse index + cache. |
| 4 | [Orchestration](./05-orchestration.md) | non-optional | Coordinate the pipeline lifecycle. The entry point that external clients (agents, CI, webhooks) talk to. |
| 5 | [Hierarchical Navigation](./06-hierarchical-nav.md) | optional | Multi-axis tree indexes over atoms (by-domain / by-team / by-lifecycle / by-tag) with synthesized labels for LLM-driven descent. |
| 6 | [Blast Radius](./07-blast-radius.md) | optional | Impact analysis for sweeping decisions. Owns blast reports + traversal indexes + iterative-refinement state. |
| 7 | [Synthesis](./08-synthesis.md) | optional | Q&A and dynamic document generation (ADRs, sequence diagrams, walkthroughs). Composes atoms, other capabilities, and external sources into a single response. Gracefully degrades to the minimum: atom traversal + LLM. |
| 8 | [LLM Gateway](./09-llm-gateway.md) | cross-cutting infrastructure | One abstraction for chat + embeddings + structured output, with per-use-case routing. Consumed by Atoms (extraction, conflict), Projection (narrative), Synthesis (composition), and Quality (LLM-as-judge). |
| 9 | [Quality](./10-quality.md) | optional, cross-cutting measurement | Benchmark + golden-set runners, faithfulness eval, LLM-as-judge harness, planted-contradiction tests, telemetry. |

Each container exposes its own read API (state + diff) and, where applicable, its own write API. There is no shared storage container — every container owns its corner of state.

## Other chapters

- [`01-principles.md`](./01-principles.md) — Data + architecture principles.
- [`11-flows.md`](./11-flows.md) — Write Path, Read Path, Blast Radius Path sequences; atom-lifecycle state machine.
- [`12-cross-cutting.md`](./12-cross-cutting.md) — Multi-tenancy, privacy boundaries, LLM call layer.
- [`13-scope-boundaries.md`](./13-scope-boundaries.md) — What's NOT in L2.

## Diagram

The L2 container diagram is generated from `likec4/L2/L2.c4`. To regenerate after changing the source: `cd likec4 && npm run export:L2`.

![L2 Containers](../diagrams/containersView.png)
