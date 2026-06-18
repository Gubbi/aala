# Recommended Pattern — Q&A and Document Generation over aala

**Status:** non-normative guidance. aala is the grounded substrate; *composition happens in the caller's agent*, not inside aala. This note records the recommended way an external agent (the caller's own LLM / skill / agent stack) turns aala's read surfaces into answers and generated documents — ADRs, white papers, marketing pitches, sequence diagrams, walkthroughs, comparisons, and so on.

## Why this is caller-side, not an aala container

aala deliberately ships no synthesis/generation container. Generation needs vary without limit (every output kind has its own structure, audience, and tooling), and callers increasingly bring capable agents of their own. Putting a generator inside aala would either constrain those agents or perpetually chase their feature set. Instead, aala exposes everything a generating agent needs to stay **grounded and navigable**, and gets out of the way of composition.

What aala provides:

- **Projection facet** (per contentful container) — readable prose + a hierarchical, summarized navigation index. See [`docs/interfaces/projection.md`](../interfaces/projection.md).
- **Structured read APIs** — precise graph queries: `traverse_relations`, `get_by_id`, `list_by_classification`, `list_by_tree`, `resolve_subject`, schema lookups. See [`docs/interfaces/atoms.md`](../interfaces/atoms.md).
- **Optional capabilities** — [Hierarchical Navigation](../interfaces/hierarchical-nav.md) (axis trees) and [Blast Radius](../interfaces/blast-radius.md) (impact reports) when wired.
- **Provenance everywhere** — every projection document and every atom carries the IDs needed to cite back to source.

## The core pattern: prose-dump-first, then graph-query-refine

The common input shape across *all* output kinds is **not** a raw dump of records, and **not** a single fixed query. It is a two-phase loop:

1. **Ground on prose.** LLMs reason markedly better over well-structured prose than over a large dump of raw atom/RDF-style records. So the agent's *initial context* is projected prose, not the atom graph. Crucially, it is not the *whole* corpus dumped flat either — it is navigated.

2. **Refine with graph queries.** Once grounded, the agent issues precise structured queries (relation traversal, schema lookups, blast reports) to pull exact facts, resolve subjects, check cardinalities, and confirm impact — the things prose alone can't guarantee.

This holds whether the agent is answering "how does checkout work?" or drafting an ADR or rendering a sequence diagram. The output kind changes the *composition* step, not the *grounding* step.

## Navigating the prose: PageIndex-style descent

Dumping every projection document defeats the purpose. The projection facet's `read_index()` returns a hierarchical tree of titled, summarized nodes. The agent:

1. Fetches `read_index()` (optionally rooted at a hint).
2. Reads node titles + one-line summaries to decide *which* branches are relevant — without fetching their full content.
3. Descends only the relevant branches, calling `read(path)` on the few leaves it actually needs.
4. Uses each document's `provenance` to jump into the structured API for precise follow-ups.

This keeps the agent's context small and on-topic, scales to large knowledge bases, and mirrors how a human skims a table of contents before reading a chapter.

## A worked recipe (illustrative)

For "draft an ADR for switching checkout to a PWA":

1. `read_index(root: "domains/payments")` → locate checkout-related documents.
2. `read(...)` the handful of relevant leaves → ground on current checkout behavior, constraints, decisions.
3. `traverse_relations` from the surfaced atoms → pull exact dependencies and constraint atoms.
4. (If wired) `Blast Radius.analyze` on the hypothetical change → impact-aware "consequences" section.
5. Compose the ADR in the agent, citing atom IDs from provenance.
6. (Optional) Propose the result back via `Atoms.propose` so the draft becomes a reviewable claim.

The same skeleton — *navigate index → read prose → graph-query → compose → optionally propose back* — produces a white paper, a pitch, a sequence diagram (compose from `traverse_relations` filtered by predicate kind), or a plain answer. Only step 5's renderer differs.

## Grounding and faithfulness

Because composition is caller-side, faithfulness checking is also primarily a caller concern: the agent should verify each asserted claim against the atoms it actually retrieved (provenance makes this mechanical). Deployments that want a managed faithfulness/quality signal can route through the [Quality](../interfaces/quality.md) capability's LLM-as-judge harness when wired; that is an evaluation surface, not a generation gate.

## What this replaces

Earlier designs placed a generation container inside aala. That responsibility now lives with the calling agent, using the surfaces above. aala's job is to make the substrate **grounded** (atoms as the system of record), **readable** (projection facet), and **navigable** (projection index + optional Hierarchical Navigation) — so any agent can compose any output kind on top.
