# 08 — Blast Radius

Blast Radius is the impact-analysis pipeline: given a proposed transition of an atom, it produces a structured report listing the atoms whose status, validity, or content would be affected. It is the human-facing surface of the cascade machinery in [07 — Atom Lifecycle](./07-atom-lifecycle.md), exposed as a queryable analysis rather than an automatic state transition.

The type shapes, method signatures, per-method **Errors** / **Access** / **Concurrency** declarations, and the event union live in [`docs/interfaces/blast-radius.md`](../interfaces/blast-radius.md) — the contract's normative home per [01 — Conventions § Normative ownership](./01-conventions.md#normative-ownership); idempotency contracts live in the registry in [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency). This chapter owns the semantics: pipeline stages, the structural-equivalence invariant, iteration, report lifecycle, determinism, and retention.

## When it runs

| Trigger | Use case |
|---|---|
| Explicit query — `analyze({ atom_id, hypothetical })` | A reviewer wants to understand the impact of a proposed change before committing to it. |
| Pre-transition preview | The Status Manager MAY invoke Blast Radius to surface predicted impact before applying a status change. |
| Ingest side-effect | Orchestration MAY run an analysis during an ingest — typically scoping the resolution of a pending situation the proposal surfaced (a flagged supersession, a hanging dependent set) — and attach the opened report's `blast_id` to the ingest summary ([`interfaces/orchestration.md § ingest`](../interfaces/orchestration.md#ingest)). Whether and when to run it is implementation-discretionary; the contract is only that `blast_id` is present iff an analysis ran. The analyzed transition must still be unapplied — analyzing an applied state yields the empty no-op set below. |
| Resolution scoping | While a report is worked, `unresolved` lists the impacts still awaiting resolution; a reviewer MAY `analyze` a contemplated resolution state for a hanging dependent before choosing it. |

A post-transition audit is **not** an analysis mode. The cascade trail of an applied transition is already recorded: the correlation-grouped `StatusChanged` / `DerivedInvalidated` events in the Atoms Delta Stream ([05 — Delta Streams](./05-delta-streams.md)) enumerate exactly what moved and why ([07 § Cascade provenance emissions](./07-atom-lifecycle.md#cascade-provenance-emissions)). `analyze` evaluates proposed transitions; analyzing a state the atom already holds is the engine's idempotent no-op ([07 § API for transitions](./07-atom-lifecycle.md#api-for-transitions)) and yields an empty structural set.

## Report anatomy

A `BlastReport` (shape in the [interface](../interfaces/blast-radius.md#types)) layers three kinds of content:

| Layer | Report fields | Nature |
|---|---|---|
| **Structural** | `impacts`, `derived_loss` | Deterministic. The cascade engine's dry-run output; covered by the structural-equivalence invariant below. |
| **Cross-tree advisory** | `cross_tree_advisories` | Deterministic. Equivalence-linked twins of impacted atoms — no predicted state, no resolution tracking. |
| **Implicit advisory** | `implicit_impacts` | LLM-inferred semantic coupling — optional, non-deterministic, refined while the report is open (§ Iteration). |

The optional `below_threshold` field lists atoms the composition confidence gate excluded (§ Confidence gating). The advisory layers live in their own fields, never inside `impacts` — that separation is what makes the structural-equivalence invariant testable.

## Pipeline stages

Blast Radius does **not** re-implement cascade traversal. The structural impact is the read-only dry-run of the one cascade engine, obtained from [`Atoms.simulate_transition`](../interfaces/atoms.md#simulate_transition). Blast Radius MUST execute these stages in order:

| Stage | What it does |
|---|---|
| **1. Origin resolution** | Resolve the input `atom_id` to a canonical atom; if it doesn't exist, fail with an `analyzing_impact` / `not_found` Activity Error ([10 — Error Model](./10-error-model.md#activity-end-states)). |
| **2. Structural cascade (delegated)** | Call `Atoms.simulate_transition(atom_id, hypothetical, { depth_limit })`. This returns the full three-channel cascade fixpoint (predicate-kind, scope-premise, derivation-invalidation) with the composition confidence gate applied — by the **same engine** that runs the real cascade ([07](./07-atom-lifecycle.md)). |
| **3. Cross-tree advisory collection** | For every atom in the structural impact set (origin included), collect each active `kind=equivalence` RelationAtom linking it to an atom in another tree. Each twin becomes a `CrossTreeAdvisory` — **advisory only**: no `predicted_state`, no `resolution_required`. Equivalence transmits no cascade ([07 § Tree boundaries and cascade](./07-atom-lifecycle.md#tree-boundaries-and-cascade)); the advisory list tells the other tree's owners which of their concepts correspond to the impacted set. |
| **4. LLM-implicit pass (optional)** | When wired, surface *non-structural* impacts an LLM infers (semantic coupling not captured by edges) as `implicit_impacts` entries — **additive and advisory**, never part of the structural set or its equivalence guarantee. |
| **5. Report assembly** | Map the simulation's impacts and invalidations into `impacts` / `derived_loss`; attach `cross_tree_advisories` and `implicit_impacts`; apply `scope_filter` to all layers at assembly (not at traversal); store the report and emit `ReportOpened`. |

The pipeline MUST NOT mutate any atom's state during analysis.

**Structural equivalence (normative).** The structural layer of every `BlastReport` — `impacts` plus `derived_loss` — MUST equal what the lifecycle cascade ([07](./07-atom-lifecycle.md)) would actually do for the same transition, because both come from the one engine via `simulate_transition`. The advisory layers are outside this equivalence. Conformance is testable by direct comparison of the structural layer with `simulate_transition`'s result. See [07 § The cascade is the single source of truth](./07-atom-lifecycle.md#the-cascade-is-the-single-source-of-truth).

## Relationship to entailment

The cascade engine walks **asserted** edges only ([07 — Atom Lifecycle § Cascade rules](./07-atom-lifecycle.md#cascade-rules)); derived edges serve queries and navigation, never propagation. Entailment still shapes the analysis:

- T1 (transitive `is_a`, `inverse_of` materialization, alias closure) — affects which atoms are reachable as classification targets, doesn't directly drive cascade.
- T2 (transitive closure of transitive kinds, chain derivation, symmetric expansion) — T2-derived atoms appear in the report only as `derived_loss`: derived atoms the transition would invalidate via channel 3. Multi-hop structural reach comes from the cascade fixpoint walking asserted edges hop by hop — an atom multiple hops away along asserted dependency edges is included through the fixpoint, never through a derived shortcut edge (no double-cascade).
- T3 (disjointness propagation, characteristic violations over asserted edges) — T3 evaluation surfaces new conflict outcomes through the conflict pipeline ([06](./06-conflict-classification.md)) when content actually changes; predicting them is not part of the structural report — the simulation predicts status transitions, not conflict classifications. Implementations MAY surface anticipated T3 violations through the advisory implicit layer.

Whether derived atoms are materialized (eager / hybrid) or computed on demand (lazy) is internal to the engine; the simulation result is the same either way.

## Confidence gating for composition

The cascade engine applies the composition confidence gate (default threshold 0.8), so `simulate_transition` already excludes below-threshold composition edges — Blast Radius does not re-apply it. The report MAY carry the excluded atoms in its optional `below_threshold` field ([interface](../interfaces/blast-radius.md#types)); implementations SHOULD populate it for reviewer awareness.

## Cross-tree behavior

Trees are autonomous existences ([03 — Data Model § Scope](./03-data-model.md#scope)); structural traversal and advisory surfacing treat them differently:

- **Asserted cross-tree edges are fully walked.** The cascade engine (via `simulate_transition`) follows RelationAtom edges regardless of tree — a `dependency` edge from a planning-tree decision to an implementation-tree constraint participates in the structural impact set like any within-tree edge.
- **`kind=equivalence` links are never walked.** Equivalence transmits no cascade ([07 § Tree boundaries and cascade](./07-atom-lifecycle.md#tree-boundaries-and-cascade)); an equivalence-linked twin in another tree is NOT part of the structural impact set and carries no `predicted_state`. Stage 3 surfaces such twins in `cross_tree_advisories` so the twin tree's owners can assess the correspondence on their own terms.
- `scope_filter` limits the *report* to specific trees; the underlying analysis still traverses asserted edges across boundaries (the filter applies at report assembly, not at traversal).

## Resolution

Under the cascade registry ([07 § Cascade rules](./07-atom-lifecycle.md#cascade-rules)), channels 1 and 2 are the only producers of structural impacts, and both hang their targets; channel-3 invalidations are automatic and appear as `derived_loss`. Every entry in `impacts` therefore carries `predicted_state: "hanging"` and `resolution_required: true`, and each is resolved **per-atom and contextually** ([07](./07-atom-lifecycle.md#cascade-rules)) — a shared trigger never implies a blanket resolution:

- **`impacts`** — a reviewer revisits each atom and either confirms the hang, defers, deprecates, restores (`active`), removes, supersedes, folds it as a duplicate, or revises it via Adapt ([07 § Adapt](./07-atom-lifecycle.md#adapt--special-case)). The choice is recorded through `record_resolution`, whose `BlastResolution` vocabulary is **closed** — exactly the legal per-atom choices, each with a defined `update_status` mapping; `rejected` is unrepresentable because committed atoms cannot be rejected ([07 § Transition table](./07-atom-lifecycle.md#transition-table)). The atom transition itself is effected through `Atoms.update_status` — the single mutation path for atom state — and **precedes** the record: a stored resolution always denotes a completed transition. Blast Radius never mutates an atom through any other route. The resolution vocabulary, the `update_status` mapping, the apply-then-record hand-off, error propagation, and concurrency are declared in [`interfaces/blast-radius.md § record_resolution`](../interfaces/blast-radius.md#record_resolution); its idempotency contract is the registry row in [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency).
- **`derived_loss`** — no human action; derived atoms are invalidated automatically (channel 3).
- **`cross_tree_advisories`** — no resolution is tracked; any follow-up on a twin is an ordinary review action taken in the twin's own tree, at its owners' discretion.
- **`implicit_impacts`** — advisory; no predicted state, no tracked resolution; any follow-up is an ordinary review action.

`record_resolution` accepts only atoms in the report's structural set — resolving an advisory entry fails with `analyzing_impact` / `not_affected`.

## Iteration

A report's structural layer is final at generation; the **implicit advisory layer** is not. While a report is open:

- Each recorded resolution is signal for the LLM-implicit pass, which MAY revise its remaining `implicit_impacts` suggestions.
- Every refinement round increments the report's `iterations` counter and emits a `ReportRefined` event whose `added` / `removed` lists name the implicit entries that changed.
- Refinement MUST NOT alter `impacts`, `derived_loss`, or `cross_tree_advisories` — the deterministic layers never change after generation.
- Refinement MUST NOT block a `record_resolution` response: it runs asynchronously off the recorded resolution (e.g., triggered by `ImpactResolved` events), and implementations SHOULD bound it — batching per several resolutions or per time window rather than re-evaluating on every call.
- Refinement stops when the report closes.

When the LLM-implicit pass is not wired, `implicit_impacts` is empty, `iterations` stays `0`, and no `ReportRefined` event is ever emitted.

## Report lifecycle

A report's `state` moves strictly forward:

- **`open`** — at generation.
- **`in_progress`** — on the first resolution recorded against the report.
- **`closed`** — when every entry in `impacts` carries a recorded resolution; `closed_at` is set. Advisory entries (`cross_tree_advisories`, `implicit_impacts`) never gate closure.

`closed` is terminal: recording a new or differing resolution against a closed report fails with `analyzing_impact` / `report_closed`. An identical re-record of an already-recorded resolution is a **replay** and succeeds as a no-op — the replay check precedes the closed check, mirroring [07 § API for transitions](./07-atom-lifecycle.md#api-for-transitions) (idempotency registry: [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency)). If circumstances change after closure, a fresh `analyze` produces a new report.

## Determinism and report identity

Given the same atom state and the same input, the deterministic layers of the report — `impacts`, `derived_loss`, `cross_tree_advisories`, `below_threshold` — MUST be identical across runs. Only `implicit_impacts` may vary.

Report identity follows from determinism:

- Re-running `analyze` with identical input against **unchanged** atom state MUST return the existing open report — same `blast_id`, no second `ReportOpened` event. "Unchanged" means no event has been appended to the Atoms Delta Stream of the bound snapshot since the report was generated.
- Once atom state has changed, a re-run generates a **fresh** report (new `blast_id`); the earlier report remains retrievable via `get_report` as a point-in-time artifact — subsequent transitions never rewrite a stored report (only § Iteration revises the implicit layer of an open one).

Implementations MAY cache reports keyed by `(atom_id, hypothetical, depth_limit, scope_filter)` within a snapshot. The RECOMMENDED staleness check is recording the Atoms Delta-Stream head `Ref` at generation and comparing it on lookup — an O(1) check that needs no invalidation machinery.

## Storage and retention

Reports are first-class: after generation they MUST be retrievable via `get_report(blast_id)`. A report MUST remain retrievable while it is `open` or `in_progress`. Closed reports MAY be garbage-collected after a deployer-configured retention window; the resolution audit trail survives the report — every resolution-driven transition is recorded on the atom itself (`status.reason`, `status.correlation_id`, and `status.meta.blast_id`, stamped by the resolution mapping in [`interfaces/blast-radius.md § record_resolution`](../interfaces/blast-radius.md#record_resolution)) and in the Delta Streams.
