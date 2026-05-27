# Projection — Atoms Rendered as Prose

**Status: non-optional** for any deployment where humans review the system's output. Atoms are the record; projections are the surface humans read.

## Concern

Projection turns the structured atom set into a structured prose set. It owns the rendered files, the indexes that map atoms to the sections they appear in, the cache that keeps rendering deterministic, and the rendering pipeline that combines template and LLM output.

In one sentence: **Projection produces the human-readable view of the canonical atom state, scoped to what changed.**

## What it owns

| Concern | Notes |
|---|---|
| Projection files | Rendered markdown, one per entity / scope (e.g., `domains/payments/services/checkout.md`, `glossary.md`, `cross-cutting/security.md`). Stored in the current snapshot alongside atoms. |
| Claim-to-section reverse index | For each atom, which projection files + section IDs it appears in. Lets a re-render scope to "only the sections atom X touches" instead of regenerating everything. |
| Render cache | Hash of `(atom_set + prompt_version + model_version)` → rendered prose. Same inputs always produce the same output. |
| Rendering pipeline | The composition of template renderer (deterministic, no LLM, for structural content) + narrative renderer (LLM, for connective prose) + previous-prose anchor (to minimize gratuitous rephrasing on re-render). |

## Why determinism matters

Without deterministic rendering, an unchanged section would re-render to slightly different prose every time, making diffs unreadable and obscuring real changes. The cache eliminates noise for unchanged sections; the anchor minimizes it for sections that *did* change. Both are part of the container's core contract, not optimizations.

## API surface (conceptual)

**Write (against the currently selected snapshot):**
- `re_render(scope?) → changes_summary` — produce or update projection files for the affected sections of the current snapshot. With no scope, re-renders everything implied by atoms that changed since the last render. Returns a summary of which projection files changed and which atoms drove the changes.

**Read:**
- `get(projection_path)` — fetch a projection file from the current snapshot.
- `affected_sections(atom_ids)` — which projection files + section IDs would change if these atoms changed. Lets callers preview the render scope without doing the work.
- `changes_since(ref)` — what changed in projections since the marker. The container's own change stream.

Projection writes to the currently selected snapshot, same as Atoms. Snapshot selection is owned by [Orchestration](./05-orchestration.md).

## Dependencies

- **[Atoms](./03-atoms.md)** — read API for the atoms that drive rendering.
- **[LLM Gateway](./09-llm-gateway.md)** — for narrative rendering. Optional internally: a template-only configuration skips the LLM path and still produces a valid (less fluent) projection.
- No dependencies on optional containers.

## Components (preview of L3)

- **Section Scoper** — uses the reverse index to determine which projection files / sections are affected by a given atom set.
- **Template Renderer** — pure function; deterministically renders structural content (tables, fact lists, schema-derived sections). No LLM.
- **Narrative Renderer** — LLM-driven; renders connective prose between structural sections. Uses the anchor.
- **Previous-Prose Anchor** — passes the previous rendering of the section to the narrative renderer with a "preserve unchanged wording" instruction.
- **Cache Manager** — looks up by hash; on hit, returns cached prose; on miss, drives the renderers and caches the result.
- **Section Assembler** — stitches template + narrative output into a final markdown section.
- **Reverse Index Maintainer** — updates the claim-to-section index as projections are written.
- **Change Annotator** — records which atoms triggered which section re-renders (consumed by clients that want to surface "what changed and why").
- **Change Log** — maintains the ordered, append-only event log for this container. Receives notifications from Markdown Writer (`SectionChanged` / `SectionAdded` / `SectionDeleted`) and from Reverse Index Maintainer (`IndexUpdated`). Serves `changes_since(ref)` and manages the ref / checkpoint surface that downstream consumers track against.

Full L3 detail lives in `docs/L3/04-projection.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| Rendering mode | Template-only (no LLM, fastest, least fluent) → template + cached LLM narrative (default) → streaming projection (incremental output for live capture). |
| Projection schema | One file per entity / service / decision (fine-grained, many small files); one file per scope (coarse-grained, fewer files); one large generated document (single-file deployments). |
| Cache backend | In-memory only; file-backed; external KV store. |
| Anchor strategy | Always anchor on the previous rendering; anchor only on high-confidence stable sections; never anchor (every re-render is fresh). |
| Glossary as a projection | The alphabetical glossary page is one of Projection's outputs — atoms of type `definition` rendered as A→Z entries with cross-references to aliases and see-also relationships. Not a separate container. |

## What Projection does NOT do

- It does not navigate atoms by query intent — that's [Synthesis](./08-synthesis.md) composing Atoms reads with the LLM.
- It does not maintain a tree for human exploration — that's [Hierarchical Navigation](./06-hierarchical-nav.md).
- It does not decide what is canonical — it only renders what Atoms says is canonical at the selected snapshot.
