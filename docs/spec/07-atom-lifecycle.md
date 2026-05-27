# 07 — Atom Lifecycle

This chapter defines the binding atom lifecycle: present states, removal outcomes, transitions, triggers, and the Hanging cascade. The narrative state-machine diagram is in [`docs/L2/11-flows.md`](../L2/11-flows.md#atom-lifecycle-state-machine); this chapter formalizes it.

## States and outcomes

An atom is at any moment in exactly one of four **present states**:

| State | Semantic |
|---|---|
| `Active` | The atom is currently held to be true. Default for new atoms. Included in active reads, projections, navigation, conflict comparisons. |
| `Hanging` | A premise the atom depended on has (or may have) changed. The atom is awaiting human resolution. Included in reads with a `Hanging` marker; flagged in conflict comparisons. |
| `Deferred` | The atom is known to be `Hanging`; the team chose to defer action. Re-surfaces in subsequent blast passes. |
| `Deprecated` | The atom holds for existing dependents; no new dependencies SHOULD be added. The Conflict pipeline MUST flag new atoms proposing dependencies on `Deprecated` atoms (see [06](./06-conflict-classification.md#deprecated-atom-interaction)). |

A removal outcome takes the atom out of the snapshot, carrying intent:

| Outcome | Required meta |
|---|---|
| `Superseded` | `meta.superseded_by` (the replacing atom) |
| `Duplicate` | `meta.duplicate_of` (the canonical atom) |
| `Rejected` | rationale (the reason) |
| `Removed` | rationale (generic; no specific intent) |

An implementation MAY apply a removal outcome as actual deletion (atom is gone) or as a tombstone status (atom remains with that status). Either is conformant; the intent MUST be recorded in the Change Log meta regardless.

## State machine

The complete transition table:

| From | To | Trigger |
|---|---|---|
| (entry) | `Active` | `Atoms.propose(fragment)` accepted as `New` / `Compatible` / `Refines` |
| (entry) | (removal) `Superseded` | Proposed atom is rejected because it Supersedes — the existing atom records `Superseded` |
| (entry) | (removal) `Duplicate` | Conflict classifies as `Duplicate` (the new atom does not enter the store) |
| (entry) | (removal) `Rejected` | Conflict classifies as `Contradicts` or `Unclear` and the implementation rejects |
| `Active` | `Hanging` | Atoms's direct-reference cascade (a referenced atom's content or premise changed). Also [Blast Radius](./08-blast-radius.md) extending the cascade via premise tags or implicit reasoning. |
| `Active` | `Deprecated` | `Atoms.update_status(atom_id, "Deprecated", rationale)` |
| `Active` | (removal) `Superseded` | A subsequently accepted atom Supersedes this one |
| `Active` | (removal) `Removed` | `Atoms.update_status(atom_id, "Removed", rationale)` — explicit non-specific removal |
| `Hanging` | `Active` | `Atoms.update_status(atom_id, "Active", rationale)` — Keep resolution |
| `Hanging` | `Active` | `Atoms.propose(...)` with revised content — Adapt resolution |
| `Hanging` | `Deferred` | `Atoms.update_status(atom_id, "Deferred", rationale)` |
| `Hanging` | `Deprecated` | `Atoms.update_status(atom_id, "Deprecated", rationale)` |
| `Hanging` | (removal) `Removed` | `Atoms.update_status(atom_id, "Removed", rationale)` |
| `Hanging` | (removal) `Superseded` | A replacement atom is accepted |
| `Deferred` | `Hanging` | [Blast Radius](./08-blast-radius.md) re-evaluation surfaces the atom again |
| `Deferred` | (removal) `Removed` | Explicit `update_status` |
| `Deprecated` | `Active` | `Atoms.update_status(atom_id, "Active", rationale)` — un-deprecation |
| `Deprecated` | `Hanging` | Reference cascade applies to a `Deprecated` atom whose premise changed |
| `Deprecated` | (removal) `Removed` | Explicit `update_status` |
| `Deprecated` | (removal) `Superseded` | A replacement atom is accepted |

Transitions NOT in this table are NOT permitted. An attempted invalid transition MUST result in `InvalidTransition` error.

## Triggers in detail

### `propose`-driven transitions

When `Atoms.propose(fragment)` runs, the Conflict pipeline (see [06](./06-conflict-classification.md)) classifies each extracted atom. Based on the classification:

- `New` / `Compatible` / `Reinforces` / `Refines` — the new atom enters as `Active`.
- `Duplicate` — the new atom is not stored; the existing atom MAY have confidence revised.
- `Supersedes` — the new atom enters as `Active`; each atom in the proposed atom's `decision_metadata.supersedes` (or otherwise identified by the judge as superseded) transitions to `Superseded`.
- `Contradicts` — the proposed atom MAY enter with `Hanging` status OR MAY be `Rejected`. The implementation's classification policy governs this choice; both MUST be possible behaviors.
- `Unclear` — the proposed atom MAY be `Rejected` OR added with low confidence. Implementation policy.

### `update_status`-driven transitions

`Atoms.update_status(atom_id, outcome, rationale, meta?)` is the only API surface for explicit transitions. The implementation MUST:

1. Verify the source-state → target-outcome transition is in the allowed table above. If not, return `InvalidTransition`.
2. For removal outcomes requiring meta (`Superseded` needs `superseded_by`; `Duplicate` needs `duplicate_of`): verify the required field is present. If not, return `MissingMeta`.
3. Apply the transition through Status Manager (or whatever the impl's mutation funnel is — see [API-only access principle](../L2/01-principles.md)).
4. Emit the corresponding Change Log event.
5. If the outcome triggers a Hanging cascade (e.g., `Superseded`), perform the cascade before returning.

### Adapt is a special case

The Adapt resolution does not have a direct `update_status` mapping. It is realized as a normal `Atoms.propose(...)` call with the revised content. The Conflict pipeline classifies the revised content (typically as `Supersedes`); the original `Hanging` atom transitions to `Superseded` with the revised atom as `superseded_by`. The revised atom enters as `Active`.

## Hanging cascade

When an atom transitions out of `Active` (to any other state or to a removal outcome), the implementation MUST cascade `Hanging` status to atoms that depended on it:

1. **Direct references (REQUIRED).** For every atom in the snapshot whose `references` or `depends_on` contains the changing atom's id, transition that atom from `Active` to `Hanging` (or leave it in its current non-`Active` state).

2. **Premise tags (RECOMMENDED if Blast Radius is wired).** For every atom whose `premises` includes a premise that the changing atom's transition invalidates, transition to `Hanging`.

3. **LLM-implicit pass (RECOMMENDED if Blast Radius is wired).** Apply the LLM-implicit analysis described in [08 — Blast Radius](./08-blast-radius.md) to identify atoms whose premises are implicitly invalidated.

Stage 1 MUST be implemented even by minimal conformant implementations. Stages 2 and 3 are part of the Blast Radius contract and are required only when Blast Radius is wired.

When an atom is cascaded to `Hanging`, the Change Log MUST emit a `StatusChanged` event with `rationale` indicating the cause (e.g., "premise atom X superseded").

## Atoms in removal outcomes

An atom with a removal outcome (whether applied as deletion or tombstone):

- MUST NOT participate in reference cascades (referencing a removed atom is a dead reference; the cascade does not flip the referencing atom unless the implementation chooses to flag dead references separately).
- MUST NOT be returned by `list_scope(scope, ["active", "hanging", "deferred", "deprecated"])` (the default present-state filter).
- MAY be returned by `list_scope(scope, ["superseded", "duplicate", "rejected", "removed"])` — implementations that apply removal as deletion return empty; implementations that tombstone return the atoms.
- MUST be reachable in the parent snapshot at the moment of transition (snapshot history is the audit trail).
