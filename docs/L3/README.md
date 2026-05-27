# L3 — Component Architecture

**Knowledge Compiler — conceptual component-level breakdown.**

This folder drills into each container from [L2](../L2/) and describes the components inside. For every container in L2, there is a corresponding chapter here. The chapters are numbered to align with L2 (e.g., `03-atoms.md` here describes the components of the container documented in `L2/03-atoms.md`).

Like L2, this folder documents the *superset* of components a full aala deployment may include. Each component carries the same optionality flag as its container; an implementation realizes whichever subset it needs.

## Chapter list

| # | Container | L2 chapter | L3 chapter |
|---|---|---|---|
| 2 | Ingestion | [L2/02](../L2/02-ingestion.md) | [02-ingestion.md](./02-ingestion.md) |
| 3 | Atoms | [L2/03](../L2/03-atoms.md) | [03-atoms.md](./03-atoms.md) |
| 4 | Projection | [L2/04](../L2/04-projection.md) | [04-projection.md](./04-projection.md) |
| 5 | Orchestration | [L2/05](../L2/05-orchestration.md) | [05-orchestration.md](./05-orchestration.md) |
| 6 | Hierarchical Navigation | [L2/06](../L2/06-hierarchical-nav.md) | [06-hierarchical-nav.md](./06-hierarchical-nav.md) |
| 7 | Blast Radius | [L2/07](../L2/07-blast-radius.md) | [07-blast-radius.md](./07-blast-radius.md) |
| 8 | Synthesis | [L2/08](../L2/08-synthesis.md) | [08-synthesis.md](./08-synthesis.md) |
| 9 | LLM Gateway | [L2/09](../L2/09-llm-gateway.md) | [09-llm-gateway.md](./09-llm-gateway.md) |
| 10 | Quality | [L2/10](../L2/10-quality.md) | [10-quality.md](./10-quality.md) |

## What an L3 chapter contains

Each chapter follows the same shape:

1. **One-line summary** of the container's concern (cross-link to its L2 chapter for the full framing).
2. **Component diagram** — rendered from `likec4/L3/L3.c4`.
3. **Component reference** — one entry per component: responsibility, internal state (if any), the events it emits or consumes, the LLM Gateway use-case keys it invokes.
4. **Internal flows** where useful — sequences across components inside the container (the cross-container sequences live in [`L2/11-flows.md`](../L2/11-flows.md)).
5. **Variation points** at the component level — which components are pluggable and how.

The canonical data model (the `Atom` class hierarchy and its supporting types) lives in [`03-atoms.md`](./03-atoms.md) since Atoms owns the schema. Other chapters cross-reference it.

## What L3 does NOT contain

- **Exact interface signatures** — those live in `docs/interfaces/` (one file per container).
- **Cross-container behavior** — that's L2.
- **Implementation-specific component choices** — those live in `docs/implementations/<impl>/L3.md`.
- **Cross-cutting concerns** — those are L2 ([`12-cross-cutting.md`](../L2/12-cross-cutting.md)).
