# Projection — The Prose Facet of Contentful Containers

**Kind: cross-cutting read facet (not a container).** **Required of [Atoms](./03-atoms.md); optional for other contentful containers.**

## Concern

Projection is the uniform, read-only surface through which a contentful container exposes a human- and LLM-readable **prose rendering** of its own state, plus a hierarchical index for navigating it. Atoms are the record; projections are the surface humans (and external agents) read.

It is deliberately **a facet, not a service**. aala does not centralize rendering into a Projection container that reaches into other containers' state — that would split one concern across the content owner and a separate renderer. Instead, each contentful container renders its *own* content, exactly as each container already owns its own read API and change stream. The coherence a single "Projection service" would offer comes instead from a shared interface contract that every implementer honors, plus the active wire profile's uniform access and access-control rules.

In one sentence: **each contentful container renders its own state to navigable prose through a common facet; there is no separate service that owns the rendered view.**

## Who implements it

| Container | Obligation | What it projects |
|---|---|---|
| [Atoms](./03-atoms.md) | **required** | Canonical claim prose + glossary — the surface humans review. |
| [Ingestion](./02-ingestion.md) | optional | Normalized raw fragments as readable source documents. |
| [Blast Radius](./07-blast-radius.md) | optional | Blast reports as readable impact narratives. |
| [Hierarchical Navigation](./06-hierarchical-nav.md) | optional | Axis trees as navigable outlines. |

## Two read surfaces, one container

A contentful container exposes two complementary read surfaces:

- **Structured read API** — precise graph queries (`traverse_relations`, `get_by_id`, `list_by_classification`, …). Exact, machine-shaped.
- **Projection facet** — readable prose + a navigable index. LLM-shaped.

These serve different consumers. An external agent answering a question grounds first on prose (LLMs reason better over prose than over raw record dumps), then drops to the structured API for precise follow-ups. See [`docs/analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md).

## The facet surface (conceptual)

Read-only, filesystem/S3-like:

- `list(prefix?)` — enumerate available projection paths.
- `read(path)` — fetch one rendered markdown document with its provenance (the atoms it was rendered from).
- `read_index(root?)` — the hierarchical, titled, summarized navigation tree (PageIndex-style) an agent descends to find the few documents it needs.
- `changes_since(ref)` — the facet's own change stream over rendered documents.

There is no write method on the facet. Rendering is private to the implementing container, driven off its own content changes. Full contract: [`interfaces/projection.md`](../../interfaces/projection.md).

## What an implementer owns behind the facet

| Concern | Notes |
|---|---|
| Rendered documents | Markdown, one per entity / scope (e.g., `domains/payments/services/checkout.md`). Stored in the current snapshot alongside the source content. |
| Navigation index | The hierarchical, summarized tree served by `read_index`. Lets a large base be consumed without dumping every record. |
| Glossary (Atoms only) | A-to-Z entries from ClassificationAtoms (concept entries), EntityAtoms classified under named-individual classifications (Person, Organization, Place, Time, Concept), and PredicateAtoms. Maintained per tree. |
| Content-to-section reverse index | For each unit of content (e.g., each atom), which documents + section IDs it appears in. Scopes a re-render to "only the sections this content touches." |
| Render cache | Hash of `(content_set + prompt_version + model_version)` → rendered prose. Same inputs always produce the same output. |
| Rendering pipeline | Template renderer (deterministic, no LLM, structural content) + narrative renderer (LLM, connective prose) + previous-prose anchor (minimize gratuitous rephrasing on re-render). |

By default, derived atoms do not drive rendered documents — projections reflect asserted claims. Implementations MAY surface derived inferences in diagnostic projections under an explicit opt-in.

## Why determinism matters

Without deterministic rendering, an unchanged document would re-render to slightly different prose every time, making diffs unreadable and obscuring real changes. The cache eliminates noise for unchanged documents; the anchor minimizes it for documents that *did* change. Both are part of the facet's contract, not optimizations.

## Dependencies

- The implementing container's own state (Atoms's atoms, Ingestion's fragments, …) — read internally.
- **[LLM Gateway](./09-llm-gateway.md)** — for narrative rendering (`aala.projection_narrative`). Optional internally: a template-only configuration skips the LLM path and still produces a valid (less fluent) document.

## Components (preview of L3)

The reference rendering pipeline a contentful container embeds: Section Scoper, Template Renderer, Narrative Renderer, Previous-Prose Anchor, Cache Manager, Section Assembler, Reverse Index Maintainer, Glossary Builder (Atoms), Index Builder, Change Annotator, Change Log. Full L3 detail lives in [`docs/L3/04-projection.md`](../L3/04-projection.md).

## Variation points (where implementations differ)

| Variation | Examples |
|---|---|
| Rendering mode | Template-only (no LLM, fastest, least fluent) → template + cached LLM narrative (default) → streaming projection (incremental output for live capture). |
| Projection schema | One document per entity / service / decision (fine-grained); one per scope (coarse); one large generated document (single-file deployments). |
| Index granularity | Flat list; per-scope outline; deep PageIndex-style tree with per-node summaries. |
| Cache backend | In-memory only; file-backed; external KV store. |
| Anchor strategy | Always anchor on previous rendering; anchor only stable sections; never anchor. |
| Glossary placement (Atoms) | One glossary document per tree; one combined cross-tree glossary; embedded into a per-domain index page. |
| Facet adoption | Atoms only (smallest conformant set); Atoms + Blast Radius; all contentful containers. |

## What Projection does NOT do

- It is not a container and owns no canonical state — every projection is derived from a content owner's state.
- It does not navigate content by query *intent* and compose answers — that is the external agent's job (see [`docs/analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md)).
- It does not maintain a tree for axis-based human exploration — that's [Hierarchical Navigation](./06-hierarchical-nav.md).
- It does not decide what is canonical — it only renders what the content owner says is canonical at the selected snapshot.
