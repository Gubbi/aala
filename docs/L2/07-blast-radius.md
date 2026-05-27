# Blast Radius — Impact Analysis for Sweeping Decisions

**Status: optional.** Deployments that don't manage sweeping decisions, or that accept the risk of unanalyzed decisions, can skip this container. Without it, a new decision atom can supersede an old one without surfacing the downstream atoms it invalidates.

## Concern

When a decision atom invalidates a premise that other atoms depended on — "use PWA instead of native Android", "deprecate the v1 API by Q3" — Blast Radius computes the set of affected atoms, categorizes them by detection mechanism, and tracks resolution as humans work through the implications.

The container owns substantial derived state (blast reports, traversal indexes, iterative refinement history) and does work no other container does (the LLM-implicit pass that finds atoms whose premises *implicitly* changed but weren't tagged). It earns its own L2 slot on both criteria.

In one sentence: **Blast Radius answers "what else changes when this decision lands?" and tracks the answer as humans resolve each affected atom.**

### Relationship to Atoms's direct-ref cascade

[Atoms](./03-atoms.md) already marks atoms `Hanging` when they directly reference an atom that Conflict has changed (Supersedes / Refines). That's the floor of impact propagation, handled without Blast Radius.

Blast Radius *extends* this floor in two ways:

1. **Premise-tag lookup** — atoms tagged with a premise that the decision changed, even if they don't directly reference the decision atom.
2. **LLM-implicit pass** — atoms that the LLM reasons no longer hold under the new premise, even without an explicit tag or reference.

When a deployment lacks Blast Radius, only the direct-ref cascade runs; tag-mediated and implicit dependents stay `Active` even though their premise has effectively changed. Blast Radius is the optional add-on that closes that recall gap.

## What it owns

| Concern | Notes |
|---|---|
| Blast reports | One report per analyzed decision. Lists affected atoms grouped by detection mechanism (direct reference, premise tag, LLM-implicit) with a suggested action per atom (keep / adapt / deprecate / remove / defer). Reports live in the current snapshot. The direct-reference rows show atoms Atoms has already marked `Hanging`; the others are atoms Blast Radius adds via the layers below. |
| Traversal indexes | Reverse indexes that make analysis fast: premise-tag → atoms; a candidate-subtree index over atoms by scope. Reference-graph traversal itself is delegated to Atoms's primitive — Blast Radius does not maintain its own reference index. |
| Iterative refinement state | As humans annotate atoms during review, the LLM-implicit pass uses those annotations as signal and re-runs against the residual set. Each blast report carries its iteration history. |
| Analysis pipeline | Premise-tag lookup → candidate subtree identification → LLM-implicit pass → transitive propagation → categorization. (Direct-ref traversal is not part of this pipeline; Atoms has already done it before Blast Radius is invoked.) |

## Resolution flow — how work returns to Atoms

A blast report doesn't end with "here are the affected atoms" — it tracks resolution until every affected atom reaches a stable state. The integration point with [Atoms](./03-atoms.md) is `record_resolution`, which translates each human decision into a change against the current snapshot:

| Human resolution | Effect in the next snapshot | What it means |
|---|---|---|
| **Keep** | `Atoms.update_status(atom_id, Active)` | The atom still holds despite the premise change. No content change. |
| **Adapt** | `Atoms.propose(...)` with the revised content | The atom's content needs revision. The human (or an agent acting for them) proposes the revised content; on acceptance it replaces the previous version through the normal extraction + conflict flow. |
| **Deprecate** | `Atoms.update_status(atom_id, Deprecated)` | The atom still holds for existing dependents, but new dependencies should not be added against it. The team is winding it down without breaking existing usage. |
| **Remove** | `Atoms.update_status(atom_id, Removed, rationale)` | The atom no longer applies under the new premise. Implementations may persist as a tombstone status or delete from the snapshot — see [`L2/11-flows.md`](./11-flows.md#atom-lifecycle-state-machine). |
| **Defer** | `Atoms.update_status(atom_id, Deferred)` | The team acknowledges the impact but chooses to act later. The atom stays present and is re-surfaced in subsequent blast passes. |

**Removal is a real delete.** No tombstoning, no terminal status. Audit and provenance come from snapshot history — the previous canonical snapshot still contains the atom, accessible by snapshot id, and the diff between snapshots shows the deletion with its blast-report rationale.

A blast report **closes** when every affected atom is in a resolved state — `Active` (kept or adapted), `Deprecated`, deleted (removed), or `Deferred`. Closed reports persist in the snapshot as a permanent record of the decision's downstream impact and the rationale for each resolution. Deferred atoms surface in periodic debt reports and are re-evaluated whenever a future blast pass touches their premise. Deprecated atoms continue to surface in conflict checks when new atoms are proposed against them.

## API surface (conceptual)

**Write (against the currently selected snapshot):**
- `analyze(decision_atom_id, options?) → blast_id, report` — run the pipeline for a decision atom. Produces a blast report and stores it in the snapshot. `options` includes pipeline depth (direct only, +premise, +implicit) and budget hints.
- `record_resolution(blast_id, atom_id, resolution, rationale)` — capture a human's decision on an affected atom (keep / adapt / remove / defer). Triggers iterative refinement: the LLM-implicit pass may revise its remaining suggestions in light of this signal. Also calls [Atoms](./03-atoms.md) `update_status` to flip the atom's status (e.g., to `Hanging`, `Adapted`, `Removed`, `Deferred`).

**Read:**
- `get_report(blast_id) → report` — fetch a full blast report.
- `list_reports(filter?)` — list reports in the current snapshot (by status, by decision, by age).
- `unresolved(blast_id?) → [affected_atom]` — flatten reports to the atoms still awaiting human resolution. Lets callers surface "what's still open" without traversing the whole report.
- `changes_since(ref)` — the container's own change stream.

## Dependencies

- **[Atoms](./03-atoms.md)** — read API (walk atoms, query by premise tags via traversal) and the `update_status` write (to mark affected atoms as `Hanging` and later as their resolution status).
- **[LLM Gateway](./09-llm-gateway.md)** — for the implicit pass and any iterative-refinement reasoning.
- No dependencies on other optional containers.

## Components (preview of L3)

- **Delta Consumer** — subscribes to Atoms's `changes_since(ref)` stream; tracks the last consumed `ref`. Filters for events relevant to blast analysis: decision-type atoms being `Added` (triggers new analyses) and `StatusChanged` events on atoms in an open report (feeds the Iterative Refiner).
- **Trigger Handler** — when the Delta Consumer surfaces a new decision-atom event, initiates a fresh blast analysis pipeline.
- **Direct-Ref Collector** — reads the atoms Atoms has already marked `Hanging` via its direct-ref cascade. These become the "direct" category in the report; Blast Radius does not re-derive them.
- **Premise-Tag Resolver** — uses the premise-tag index to find atoms tagged with the now-changed premise.
- **Subtree Identifier** — bounds the candidate set for the implicit pass (don't scan unrelated entities).
- **LLM-Implicit Pass** — for each candidate atom, asks: "does this still hold under the new premise?" Batched aggressively.
- **Transitive Propagator** — re-runs the pipeline on newly-affected atoms until a fixed point.
- **Categorizer** — groups affected atoms by detection mechanism (direct / premise / implicit) and by suggested action (keep / adapt / deprecate / remove / defer).
- **Report Builder** — produces the persisted blast report.
- **Iterative Refiner** — receives `record_resolution` calls; updates the report and re-runs the implicit pass with new signal.
- **Change Log** — maintains the ordered, append-only event log for this container. Receives `ReportOpened` / `AffectedAtomResolved` / `ReportRefined` / `ReportClosed` notifications from Report Builder and Iterative Refiner. Serves `changes_since(ref)` for [Synthesis](./08-synthesis.md) (impact-aware answers) and [Quality](./10-quality.md) (impact-detection telemetry).

Full L3 detail lives in `docs/L3/07-blast-radius.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| Pipeline depth | Direct refs only (fast, low recall); +premise tags (medium); +LLM-implicit pass (full coverage). Each deployment chooses its cost-vs-recall point. |
| Implicit-pass model tier | Top-tier LLM (highest fidelity, highest cost); mid-tier with self-check; small local model for privacy-constrained tenants. |
| Iteration policy | Single-pass (analyze once, never refine); fixed iterations (e.g., re-run twice); iterative until fixed point (default for full-coverage impls). |
| Subtree scoping | Conservative (large candidate set, more LLM calls); aggressive (tighter scoping, may miss). |
| Report persistence | In the snapshot alongside atoms (default); container-internal only; emitted as events. |

## What Blast Radius does NOT do

- Detect that an atom is a *decision* type or that it supersedes another — that's [Atoms](./03-atoms.md) (Conflict).
- Mutate atoms beyond status transitions — content edits (adapting an affected atom's payload) are made by humans against the atom, not by Blast Radius.
- Communicate the report to humans — the agent or UI reads reports via the read API and presents them. Blast Radius only produces and tracks; presentation lives outside.
- Decide *whether* to act on the report — humans (or higher-level orchestration) drive the resolution loop; Blast Radius records the resolutions and refines.
