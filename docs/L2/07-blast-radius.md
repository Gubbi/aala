# Blast Radius — Impact Analysis for Transitions

**Status: optional.** Deployments that don't need impact-analysis-as-a-service can skip this container. Without it, the cascade machinery in Atoms (per [`docs/spec/07-atom-lifecycle.md`](../spec/07-atom-lifecycle.md)) still runs on every transition — Blast Radius doesn't *cause* the cascade, it *surfaces* it as a queryable read-only report.

## Concern

Blast Radius computes the set of atoms affected by a transition (actual or hypothetical) and tracks resolution as humans work through the report. Given an atom and a target status, it produces a structured report listing every atom whose status would change, the cascade path that reaches it, and the reviewer's resolution path.

The container owns substantial derived state (blast reports, traversal indexes, iterative refinement history) and exposes the cascade machinery as a queryable analysis. It earns its own L2 slot.

In one sentence: **Blast Radius answers "what else changes when this atom transitions?" and tracks the answer as humans resolve each impacted atom.**

## Relationship to the cascade in Atoms

Atoms already runs a cascade fixpoint on every transition through three channels (per [`docs/spec/07-atom-lifecycle.md`](../spec/07-atom-lifecycle.md)):

1. **Predicate-kind cascade** — dependency / composition rules per RelationAtom's kind.
2. **Scope-premise cascade** — every atom whose `Scope.constraint_atoms` contains the changing atom transitions to `hanging`.
3. **Derivation invalidation** — every derived atom whose `derivation.from` contains the changing atom is invalidated.

Blast Radius **walks the same channels** but as a read-only analysis. It can run **before** an actual transition (the `hypothetical` mode) to preview impact, or **after** a transition to audit the cascade trail. The optional LLM-implicit pass extends the analysis to atoms whose impact is implicit (not captured by the three structural channels).

When Blast Radius is absent, the cascade still runs — Atoms handles it. The Blast Radius value-add is the **report** (queryable, reviewable, resolution-trackable) and the **implicit pass** that surfaces non-structural impacts.

## What it owns

| Concern | Notes |
|---|---|
| Blast reports | One report per analysis. Lists impacted atoms with cascade paths, predicted states, trees affected, and resolution tracking. Reports live in the current snapshot or container-internal storage (impl choice). |
| Structural impact (delegated) | The structural impact set comes read-only from [`Atoms.simulate_transition`](../interfaces/atoms.md) — the dry-run of the one cascade engine. Blast Radius owns **no** cascade-traversal logic or reverse indexes of its own; that lives in Atoms, guaranteeing the report matches the real cascade. |
| Iterative refinement state | As humans annotate atoms during review, the LLM-implicit pass uses those annotations as signal and re-runs against the residual set. Each report carries its iteration history. |
| Analysis pipeline | The pipeline from [`docs/spec/08-blast-radius.md`](../spec/08-blast-radius.md): origin resolution → structural cascade via `Atoms.simulate_transition` → optional LLM-implicit pass (additive) → report assembly. |

## Resolution flow — how work returns to Atoms

A blast report doesn't end with "here are the impacted atoms" — it tracks resolution until every impacted atom reaches a stable state. The integration point with [Atoms](./03-atoms.md) is `record_resolution`, which translates each reviewer decision into a status transition on the affected atom via `Atoms.update_status`. Adapt (revising a hanging atom) is handled by the caller separately invoking `Atoms.propose(...)` with revised content.

A blast report **closes** when every impacted atom is in a resolved state. Closed reports persist as a permanent record of the transition's downstream impact and the rationale for each resolution.

## API surface (conceptual)

**Write (analysis is read-only against Atoms; record_resolution mutates via Atoms):**
- `analyze(input) → blast_report` — compute the impact set. `input.atom_id` is required; `input.hypothetical` lets the analysis preview a proposed transition; `input.depth_limit` and `input.scope_filter` bound the traversal.
- `record_resolution(blast_id, atom_id, resolution, rationale, meta?) → updated_report` — capture a reviewer's decision on an impacted atom. Calls `Atoms.update_status` to apply the resolution. Triggers iterative refinement: the LLM-implicit pass may revise its remaining suggestions in light of this signal.

**Read:**
- `get_report(blast_id) → report`
- `list_reports(filter?)`
- `unresolved(blast_id?) → [impact]` — flatten reports to the atoms still awaiting reviewer resolution.
- `changes_since(ref)` — the container's own change stream.

## Dependencies

- **[Atoms](./03-atoms.md)** — `simulate_transition` (the read-only cascade dry-run that produces the structural impact), other read APIs, and `update_status` (to apply resolutions). The three cascade channels live in Atoms and are **not** re-implemented here.
- **[LLM Gateway](./09-llm-gateway.md)** — for the optional implicit pass (`aala.blast_implicit`) and any iterative-refinement reasoning.
- No dependencies on other optional containers.

## Components (preview of L3)

- **Delta Consumer** — subscribes to Atoms's `changes_since(ref)` stream; filters for transition events relevant to open reports (feeds Iterative Refiner).
- **Trigger Handler** — when a high-impact transition is detected (sweeping decision, classification change with broad downstream), MAY initiate a fresh analysis automatically.
- **Cascade Simulation** — obtains the structural impact set from `Atoms.simulate_transition` (the read-only dry-run of the one cascade engine: three channels + cross-tree equivalence + composition gate, run to fixpoint). Does not re-implement the traversal.
- **LLM-Implicit Pass** — for each candidate atom (bounded by subtree scoping), asks: "does this still hold under the proposed transition?" Batched aggressively. Additive and advisory — outside the structural guarantee.
- **Report Builder** — produces the persisted blast report.
- **Iterative Refiner** — receives `record_resolution` calls; updates the report and re-runs the implicit pass with new signal.
- **Change Log** — maintains the ordered, append-only event log for this container.

Full L3 detail lives in [`docs/L3/07-blast-radius.md`](../L3/07-blast-radius.md).

## Variation points (where implementations differ)

| Variation | Examples |
|---|---|
| Pipeline depth | Structural channels only (fast, low recall); + LLM-implicit pass (full coverage). |
| Implicit-pass model tier | Top-tier LLM (highest fidelity); mid-tier with self-check; small local model for privacy-constrained tenants. |
| Iteration policy | Single-pass (analyze once, never refine); fixed iterations; iterative until fixed point (default for full-coverage implementations). |
| Subtree scoping | Conservative (large candidate set, more LLM calls); aggressive (tighter scoping, may miss). |
| Report persistence | In the snapshot alongside atoms; container-internal only; emitted as events. |
| Composition threshold gating | Default 0.8 (per spec); deployment-overridable. Affects which composition cascades are walked. |

## What Blast Radius does NOT do

- Cause cascades — that's Atoms's Status Manager (per [`docs/spec/07-atom-lifecycle.md`](../spec/07-atom-lifecycle.md)). Blast Radius is read-only analysis.
- Detect conflicts — that's Atoms's Conflict pipeline.
- Mutate atoms — except via `Atoms.update_status` invoked from `record_resolution`. Content edits (adapting an impacted atom's payload) are made by humans against the atom via `Atoms.propose`, not by Blast Radius.
- Communicate the report to humans — the agent or UI reads reports via the read API and presents them.
- Decide *whether* to act on the report — humans drive the resolution loop; Blast Radius records the resolutions and refines.
