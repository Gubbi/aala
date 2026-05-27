# 03 — Data Model

This chapter defines the binding atom data model. The full shape lives in [`docs/interfaces/00-shared-types.md`](../interfaces/00-shared-types.md); this chapter adds the normative semantics.

## Atom

An **atom** is a structured claim. Every atom MUST have:

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Globally unique within a snapshot. Stable across snapshots — the same logical atom MUST keep the same `id` through any number of `Updated` events. |
| `type` | `AtomType` | yes | One of the registered atom types (see below). |
| `subject` | string | yes | The thing the claim is about. |
| `predicate` | string | yes | The relation. |
| `object` | string | yes | The thing the claim asserts about the subject. |
| `attributes` | map | yes | Free-form structured metadata. MAY be empty. |
| `status` | `AtomStatus` | yes | One of the four present states. See [07 — Atom Lifecycle](./07-atom-lifecycle.md). |
| `status_reason` | string | no | Human-readable rationale for the current status. |
| `status_changed_at` | ISO 8601 timestamp | conditional | REQUIRED when the atom is not in its initial `Active` state. |
| `premises` | list of `Premise` | yes | MAY be empty. Premise tags drive the Hanging cascade ([07](./07-atom-lifecycle.md)) and Blast Radius ([08](./08-blast-radius.md)). |
| `sources` | list of `Source` | yes | Provenance back to the originating fragments. MUST be non-empty for atoms not created by Refines or Adapt — i.e., every atom MUST trace back to at least one fragment. |
| `confidence` | number | yes | Range 0.0–1.0. Conflict pipeline assigns; Adapt-driven content updates may revise. |
| `confidence_factors` | map | yes | Breakdown of contributing signals. MAY be empty in simple implementations. |
| `scope` | `Scope` | yes | Environment, time bounds, conditional applicability. |
| `references` | list of `AtomId` | yes | Outgoing free-form references. |
| `depends_on` | list of `AtomId` | yes | Outgoing dependency edges (used by Hanging cascade). |

Implementations MAY add additional fields to atoms. Additional fields MUST be carried through `Added` / `Updated` events and MUST NOT change the meaning of the listed fields.

## Atom types

The following atom types are part of the `aala-v1` registered taxonomy. A conformant implementation MUST support every listed type:

| Type | Meaning |
|---|---|
| `decision` | A choice made (or proposed). MAY use the `DecisionAtom` subtype with `rationale`, `supersedes`, `decision_metadata`. |
| `definition` | A term and its meaning. MAY use the `DefinitionAtom` subtype with `aliases`, `see_also`, `canonical_for`. |
| `dependency` | "X depends on Y." Outgoing edge represented via `depends_on`. |
| `constraint` | A limit, invariant, or non-functional requirement. |
| `behavior` | A described system behavior under defined conditions. |
| `capability` | A described system capability. |
| `scenario` | A described usage scenario or flow. |

Implementations MAY register additional atom types for deployment-specific domains. Additional types MUST be opaque to all containers that don't recognize them — i.e., they MUST be readable, traversable, and storable but MAY be skipped by extractors and judges that don't know how to interpret them.

## Atom subtypes

`DecisionAtom` and `DefinitionAtom` are formally defined subtypes. An atom whose `type` is `decision` or `definition` MAY conform to the corresponding subtype shape (with the additional fields). Implementations MUST treat an atom that lacks subtype-specific fields as a base atom (subject/predicate/object only); the subtype fields are optional enrichments.

For `DefinitionAtom`:

- `aliases` (list of string) — equivalent surface forms for `subject`. The Conflict pipeline ([06](./06-conflict-classification.md)) MUST consider `aliases` when classifying new atoms against existing definitions.
- `see_also` (list of `AtomId`) — related concepts. Surfaced in projections; never used for resolution.
- `canonical_for` (string) — optional reverse pointer marking this atom as the canonical name when multiple definitions overlap.

For `DecisionAtom`:

- `rationale` (string) — why the decision was made.
- `supersedes` (list of `AtomId`) — atoms this decision supersedes. When present and non-empty, the Conflict pipeline MUST emit `Superseded` outcomes for each listed atom.
- `decision_metadata` (map) — free-form decision context (alternatives considered, scope, etc.).

## Atom status

Atom status is one of four present states or four removal outcomes. Detailed transition rules are in [07 — Atom Lifecycle](./07-atom-lifecycle.md). Present states:

- `Active` — currently held. The default.
- `Hanging` — premise may have changed; awaiting resolution.
- `Deferred` — known to be hanging; team chose to defer.
- `Deprecated` — still holds for existing dependents; no new dependencies should be added.

Removal outcomes (apply when the atom transitions out of the snapshot, with intent carried in `meta` and recorded in the Change Log):

- `Superseded` — replaced by another atom (`meta.superseded_by` REQUIRED).
- `Duplicate` — was a duplicate of a canonical atom (`meta.duplicate_of` REQUIRED).
- `Rejected` — proposal rejected; never made it to canonical.
- `Removed` — generic removal.

Implementations MAY apply removal outcomes as actual deletion (atom is gone from the snapshot; intent in Change Log) OR as a tombstone status on the atom (atom remains with the removal-outcome status). Both are conformant.

## Provenance: `Source`

Every atom's `sources` list MUST trace back to at least one fragment (raw input) when the atom was extracted from a fragment. An atom created by Adapt (revising a `Hanging` atom) MAY inherit sources from the prior atom and append the new source.

```typescript
interface Source {
  uri:            string  // source artifact URI
  quote:          string  // the supporting text from the source
  extracted_at:   string  // ISO 8601
}
```

## Scope

```typescript
interface Scope {
  environment?:  string
  time_scope?:   TimeScope
  conditions:    string[]  // free-form predicates; MAY be empty
}

interface TimeScope {
  effective_from?: string  // ISO 8601
  effective_to?:   string  // ISO 8601
}
```

An atom with `time_scope.effective_to < now()` MAY be filtered from active reads by the implementation. The atom remains in the snapshot regardless.

## Schema extensibility

The taxonomy of atom types, the set of atom fields, and the status enum are versioned per the rules in [11 — Versioning](./11-versioning.md). Patch-compatible additions (new optional fields, new atom types, new attributes) MUST NOT break implementations that don't recognize them.
