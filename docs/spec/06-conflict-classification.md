# 06 — Conflict Classification

When a new atom is proposed, Atoms's Conflict pipeline MUST classify it against the canonical store. This chapter defines the eight classification outcomes and their semantics.

## Outcomes

| Outcome | Semantic |
|---|---|
| `New` | No existing canonical atom corresponds to the proposed atom. |
| `Compatible` | An existing canonical atom is similar but the new atom adds nothing — it's compatible co-existence. Both atoms remain in the snapshot. |
| `Duplicate` | The proposed atom is semantically equivalent to an existing canonical atom under the same scope. The proposed atom MUST NOT be added. |
| `Reinforces` | The proposed atom agrees with an existing canonical atom and provides additional evidence. The existing atom's `confidence` MAY be raised; provenance MAY be augmented. The proposed atom MAY be added as a sibling or merged into the existing atom (implementation choice). |
| `Refines` | The proposed atom is a more specific instance of an existing canonical atom (e.g., a constraint atom that narrows a more general one). The existing atom remains; the new atom is added with a `references` link to the original. |
| `Supersedes` | The proposed atom replaces an existing canonical atom. The existing atom MUST be transitioned to the `Superseded` removal outcome with `meta.superseded_by = <new atom id>`. |
| `Contradicts` | The proposed atom is incompatible with an existing canonical atom. Both MAY be retained pending human resolution; the new atom's status MAY be `Hanging` or its outcome MAY be `Rejected`. |
| `Unclear` | The pipeline cannot confidently classify. The atom's outcome MAY be `Rejected` (with `Unclear` recorded in the meta) or it MAY be added with low confidence pending human review. |

## Pipeline stages

The Conflict pipeline MUST implement at least the deterministic stage. The other stages are RECOMMENDED:

1. **Deterministic match** (REQUIRED) — Field-level comparison against canonical atoms with the same scope. Detects `New` (no match), `Duplicate` (exact match after normalization), and explicit `Supersedes` (when the proposed atom's `decision_metadata.supersedes` references a known atom). Other outcomes MUST NOT be emitted by this stage alone.

2. **Embedding-NN candidate finding** (RECOMMENDED) — Vector similarity search over canonical atom embeddings to surface candidate matches that deterministic comparison would miss (paraphrasing, alias use, etc.). Produces a shortlist for stage 3.

3. **LLM judge** (RECOMMENDED) — Pairwise classification of the proposed atom against each shortlist candidate. Emits any of the eight outcomes with rationale.

A conformant implementation that omits stages 2 and 3 produces low-recall classifications — `Duplicate` and `Supersedes` cases requiring semantic understanding will be missed. The implementation MUST declare which stages it implements via `Atoms.capabilities()` extension fields (implementation-defined; the standard `aala.capabilities()` returns container-level only).

## Alias resolution

A `DefinitionAtom`'s `aliases` list MUST be considered when classifying new atoms. Specifically:

- A proposed `DefinitionAtom` with `subject = X` MUST be checked for `Duplicate` against any canonical atom where `X` ∈ `atom.aliases` or `X = atom.subject`.
- A new atom that references a `DefinitionAtom` (via `references` or by quoting an alias as its subject) MUST resolve the reference through the alias graph.

The exact resolution mechanism is implementation-specific (the Conflict pipeline MAY use an alias-only deterministic stage, MAY use embeddings, MAY use the LLM judge). The contract is that alias-equivalent definitions MUST NOT result in `New` outcomes — they MUST classify as `Duplicate` or `Refines`.

## Deprecated atom interaction

When a new atom is proposed with a reference (`references` or `depends_on`) to a `Deprecated` atom, the Conflict pipeline MUST surface this for review. The proposed atom MAY be:

- Added with a warning attached to its `attributes` (implementation-specific format), OR
- Classified as `Unclear` with rationale citing the deprecated dependency.

In both cases, an event of kind `Updated` or `Added` MUST appear on the Atoms change stream so consumers can react.

## Decision atoms

A `decision`-type atom with a non-empty `supersedes` list MUST cause the Conflict pipeline to:

1. For each `atom_id` in `supersedes`:
   - Verify the atom exists in canonical state. If absent, classify the decision atom as `Unclear`.
   - Otherwise, classify the existing atom as `Superseded` and transition it via `Atoms.update_status(atom_id, "Superseded", rationale, { superseded_by: <new decision atom id> })`.

2. Trigger the **Hanging cascade** (see [07 — Atom Lifecycle](./07-atom-lifecycle.md)) for atoms with direct references to the superseded atoms.

3. If [Blast Radius](./08-blast-radius.md) is wired, trigger `analyze(decision_atom_id)` to extend the cascade.

## Outcome events

Each classification outcome maps to specific Atoms change-stream events:

| Outcome | Events emitted |
|---|---|
| `New` | `Added` (new atom) |
| `Compatible` | `Added` (new atom). The existing atom is NOT modified. |
| `Duplicate` | NONE for the proposed atom (it's not added). Optionally, `Updated` on the existing atom if `confidence` is revised. |
| `Reinforces` | Either `Added` (sibling) or `Updated` (existing atom). |
| `Refines` | `Added` (new atom with reference to original). |
| `Supersedes` | `Added` (new atom) + `Superseded` (existing atom). |
| `Contradicts` | `Added` (new atom, possibly with status `Hanging`) OR `Rejected` (new atom). |
| `Unclear` | `Added` with low confidence OR `Rejected`. |

## Rationale capture

Every classification MUST capture a rationale string. For LLM-judge classifications, the rationale SHOULD be the judge's reasoning. For deterministic classifications, the rationale MAY be a short canned string ("field-level match," "explicit supersedes").

The rationale is exposed via `ProposeResult.classifications[].rationale` and recorded in the corresponding Change Log event.

## Confidence assignment

A new atom added via Conflict MUST have a `confidence` value set:

- `New` atoms: inherit confidence from extraction (per the Extractor's signals).
- `Reinforces` atoms (when added as siblings): inherit and MAY be raised.
- `Refines` atoms: inherit from extraction.
- Other outcomes: inherit; values MAY be capped by implementation policy.

`confidence_factors` SHOULD record at least the contributing signals (e.g., `{ "extraction": 0.8, "conflict_corroboration": 0.1 }`).
