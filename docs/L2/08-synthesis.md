# Synthesis — Q&A and Dynamic Document Generation

**Status: optional.** A deployment can skip Synthesis and still serve queries by giving callers raw read access to Atoms (and Hierarchical Navigation if present). Synthesis exists to compose multiple capabilities into a single response: synthesized answers, generated ADRs, sequence diagrams, walkthroughs, comparisons.

## Concern

Synthesis takes an intent ("how does checkout work?", "draft an ADR for switching to PWA", "what changes if we deprecate the v1 API?") and composes the available capabilities into a single response with provenance. The container's value is the composition itself — picking the right generator, deciding which capabilities to consult, assembling outputs, attaching citations, checking faithfulness.

In one sentence: **Synthesis is the container that turns "I want to know / produce X" into a response, by orchestrating reads across whichever capabilities are wired up.**

## What it owns

| Concern | Notes |
|---|---|
| Generator registry | Pluggable generators, one per output kind: Q&A, ADR, sequence diagram, walkthrough, comparison, etc. Each generator declares what capabilities it requires (the floor) and which it optionally uses (graceful enhancement). |
| Intent → generator routing | Classify the incoming intent and pick a generator. May be rule-based, LLM-classified, or both. |
| Capability composition | For the chosen generator, decide which capabilities to consult and in what order. Adapts at runtime to which capabilities are present in this deployment. |
| Faithfulness gate | Post-generation check: every assertion in the response must be supported by atoms actually retrieved. Catches confabulation. |
| Provenance attachment | Every response includes citations: atom IDs, navigation paths, sources consulted. |
| Optional response cache | Hashed by `(intent + atom set + generator version + model)` — same inputs, same response. Optional because not all impls want to cache. |

Synthesis does not own canonical state. Generated output is *proposed*; promoting it to canonical (e.g., an ADR draft becoming a committed decision atom) goes through the normal [Atoms](./03-atoms.md) `propose` path, not through a Synthesis-private write.

## Graceful degradation — the floor and the enhancements

Every generator declares its capability requirements as **floor + optional**.

| Layer | Capability | What it gives the generator |
|---|---|---|
| Floor | Atoms read API | Atom-level facts. The minimum any generator needs. |
| Floor | LLM Gateway | The LLM call that turns retrieved atoms into prose / diagrams. |
| Optional | Hierarchical Navigation | Intent-driven tree descent instead of brute-force atom traversal. Sharper retrieval, fewer false positives. |
| Optional | Blast Radius | Impact-aware answers: "if we deprecate X, here's what's hanging." |
| Optional | Projection | Style continuity — anchor generated prose against the existing rendered docs so voice stays consistent. |
| Optional | External data sources | Web search, vendor MCPs, internal knowledge bases. Lets a generator pull in context that isn't in the atom set yet. |

When an optional capability is absent, the generator falls back to its floor and produces a lower-fidelity but still valid response. Synthesis surfaces *what it had to work with* in the response's provenance — a generator that ran without Hierarchical Navigation says so, so the caller knows the response is brute-force atom-traversal-quality.

## API surface (conceptual)

**Read:**
- `generators() → [generator_meta]` — list of available generators with their declared requirements and current operability (based on which capabilities are wired up in this deployment).
- `query(intent, generator?, hints?) → response_with_provenance` — main entry. If `generator` is omitted, intent classification picks one. Response includes the synthesized content, the atoms / sources it drew from, and the navigation path / capability list consulted.
- `changes_since(ref)` — for impls that track query history; trivial / empty for stateless impls.

Synthesis exposes no write API. Generated content is returned to the caller; if the caller wants to persist it (e.g., commit an ADR draft), they call `Atoms.propose` with the synthesized content as a fragment.

## Dependencies

- **[Atoms](./03-atoms.md)** — read API. The floor.
- **[LLM Gateway](./09-llm-gateway.md)** — the synthesis itself. The floor.
- **[Hierarchical Navigation](./06-hierarchical-nav.md)** — soft. Used by generators that want indexed descent; absence means falling back to Atoms traversal.
- **[Blast Radius](./07-blast-radius.md)** — soft. Used by generators producing impact-aware output; absence means impact-blind responses.
- **[Projection](./04-projection.md)** — soft. Used by generators that anchor to existing prose style; absence means generators write from scratch.
- **External data sources** — soft, per-deployment. May include web search MCPs, vendor doc MCPs, internal knowledge bases.

Optional-on-optional is fine here because each generator declares its floor and degradation behavior. The principle of "core capabilities must not require optional ones" is satisfied: Synthesis itself is optional, and it can degrade smoothly.

## Components (preview of L3)

- **Intent Parser** — classifies the incoming intent into a generator kind. May be LLM-driven or rule-based.
- **Generator Registry** — pluggable generators: `qa`, `adr`, `sequence_diagram`, `walkthrough`, `comparison`, plus any deployment-specific ones.
- **Capability Composer** — for a chosen generator, decides which capabilities to invoke and in what order based on what's wired up.
- **Audience Adapter** — adjusts framing for PM / Designer / Engineer / Exec audience.
- **Conflict Surfacer** — detects unresolved conflicts in the retrieved atoms; expresses them in the response ("current view: X / conflicting evidence: Y") rather than silently picking a side.
- **Confidence Temperer** — adjusts language certainty based on retrieved-atom confidence scores.
- **Provenance Attacher** — attaches atom IDs, navigation paths, sources as citations.
- **Faithfulness Checker** — verifies every claim in the response is grounded in retrieved atoms. Configurable: soft warning vs. hard block.
- **Diagram Generator** — for structural output (sequence diagrams, C4-style diagrams), composes from atoms' references and generates a rendered form (mermaid, likec4, etc.).

Full L3 detail lives in `docs/L3/08-synthesis.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| Generator set | Q&A only (smallest) → +ADR → +sequence diagram → +walkthrough → +comparison → +deployment-custom generators. Each is independently pluggable. |
| Intent classification | Rule-based (cheap, brittle); LLM-driven (more accurate, more expensive); hybrid. |
| Faithfulness mode | Off (no check); warn (return response with confabulation flag); block (fail the request if any claim is unsupported). |
| Audience adaptation | Single voice; per-role tuned (PM / Designer / Engineer / Exec). |
| External-source set | None; web search; vendor MCPs; internal company sources. |
| Response cache | None; in-process; persistent. |

## What Synthesis does NOT do

- Build navigation indexes — that's [Hierarchical Navigation](./06-hierarchical-nav.md).
- Render canonical projections — that's [Projection](./04-projection.md).
- Mutate atoms — Synthesis is read-only. Generated output becomes canonical only when the caller proposes it via [Atoms](./03-atoms.md).
- Decide what is true — atoms are the canonical statement of truth; Synthesis composes from them and surfaces conflicts rather than resolving them.
