# 08 — Blast Radius

This chapter is normative for implementations that wire the Blast Radius container. Implementations without Blast Radius are conformant; they MUST simply omit the extended Hanging cascade described here.

## Trigger conditions

Blast Radius analysis MUST be triggered when:

1. An atom with `type = "decision"` is added or transitions states in a way that invalidates a premise that other atoms depend on. Specifically:
   - The decision atom's `decision_metadata.supersedes` list is non-empty, OR
   - The Conflict pipeline classifies the decision atom as `Supersedes` against some existing canonical atom, OR
   - The decision atom transitions to `Superseded` (in which case the *replacing* decision's analysis encompasses this).

Blast Radius MAY be triggered explicitly via `BlastRadius.analyze(decision_atom_id)` regardless of automatic detection.

## Pipeline stages

The analysis pipeline has three stages. The minimum conformant Blast Radius implementation runs stage 1; stages 2 and 3 are RECOMMENDED.

### Stage 1 — Direct-reference traversal (REQUIRED)

For the decision atom being analyzed, walk the inverse `references` and `depends_on` graphs to find every atom whose explicit relationship points at the decision atom or at an atom the decision atom supersedes.

The result MUST be included in the blast report with `detection = "direct"`.

### Stage 2 — Premise-tag lookup (RECOMMENDED)

For each premise tag invalidated by the decision atom's change (e.g., `platform: android` when the decision is "use PWA instead of Android"), query the Premise-Tag Index for atoms carrying that tag.

The decision atom MUST declare which premise tags it invalidates. The mechanism for declaration is implementation-specific (a field on the decision atom, a separate annotation, or judge-derived). The contract is: the Premise-Tag Index has a defined input — "which tags does this decision change?" — and returns the atoms affected.

The result MUST be included in the blast report with `detection = "premise_tag"`.

### Stage 3 — LLM-implicit pass (RECOMMENDED)

For each atom in a bounded candidate subtree (impl-defined), invoke the LLM Gateway with `aala.blast_implicit` to ask: "given that decision X has changed, does atom Y still hold?"

The candidate subtree MUST be bounded to keep cost feasible. Common bounding rules: the same scope path, the same domain tree, atoms with premise tags overlapping the decision's. The exact bounding rule is implementation choice.

The result MUST be included in the blast report with `detection = "llm_implicit"`.

## Status cascade

For each atom identified by stages 1–3, the analysis MUST flip the atom's status from `Active` to `Hanging` via `Atoms.update_status(atom_id, "Hanging", rationale, { blast_id })`. The rationale MUST cite the originating decision atom.

Atoms already in `Hanging` or `Deferred` remain in that state but MUST be re-categorized in the new blast report. Atoms in `Deprecated` MUST also transition to `Hanging` (per the lifecycle table in [07](./07-atom-lifecycle.md)).

Atoms in removal outcomes (already removed) MUST NOT appear in a new blast report.

## Categorization

The blast report MUST group affected atoms by `detection` (which stage found them) and by `suggested_action`:

| `suggested_action` | Heuristic |
|---|---|
| `Keep` | Default for atoms whose dependence on the changed premise is incidental. |
| `Adapt` | Default for atoms whose content needs revision (e.g., refers to the superseded thing by name). |
| `Deprecate` | Default for atoms with dependents — a graceful winding-down is appropriate. |
| `Remove` | Default for atoms whose only purpose was supporting the now-superseded path. |
| `Defer` | Default for atoms with high uncertainty about the right resolution. |

The heuristic source is implementation-specific (rule-based, LLM-suggested, etc.). Human review may override the suggestion at resolution time. The contract is that *every* affected atom has *some* suggestion; it MUST NOT be left blank.

## Iterative refinement

When a human resolves an affected atom via `BlastRadius.record_resolution(blast_id, atom_id, resolution, rationale)`:

1. The corresponding `Atoms.update_status` (or `Atoms.propose`, for Adapt) call MUST be issued.
2. The blast report MUST be updated to reflect the resolution.
3. If iterative refinement is enabled, the LLM-implicit pass MAY re-run against the remaining unresolved set, using the new resolution as signal (e.g., "the team confirmed atom X is still valid; what does that tell us about atoms Y, Z?").
4. If re-running produces a different set of unresolved atoms, the new set MUST be captured in a `ReportRefined` event.

Iteration MUST terminate. Implementations MUST cap the number of refinement passes (a `max_iterations` policy is reasonable) to prevent unbounded loops.

## Report lifecycle

A blast report progresses through states:

| State | Meaning |
|---|---|
| `open` | New report; no resolutions yet. |
| `in_progress` | At least one resolution recorded; some affected atoms still unresolved. |
| `closed` | Every affected atom is in a resolved state (`Active`, `Deferred`, `Deprecated`, or a removal outcome). |

A `ReportClosed` event MUST be emitted when the report transitions to `closed`. The closed report MUST persist in the snapshot as a permanent record.

## Cross-snapshot persistence

Blast reports are snapshot-bound. A report opened in snapshot `S` MUST be reachable from `S` and its descendants. If snapshot `S` is published as canonical, the report becomes part of canonical history.

A report MAY be re-evaluated against a later snapshot if the premise tags change again; this MAY produce a new report (new `blast_id`) rather than mutating the existing one. The implementation chooses; both are conformant.

## External commitments

A `decision_metadata.external_commitment = true` flag on a decision atom MAY trigger additional behaviors: notifying additional reviewers, blocking auto-application of cascades, etc. The flag is part of the registered taxonomy; specific behaviors are implementation choices.
