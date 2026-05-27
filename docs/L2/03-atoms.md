# Atoms — The Canonical Knowledge Record

**Status: non-optional.** Every aala deployment includes this container; everything else either feeds it or reads from it.

## Concern

Atoms is the System of Record. It owns the canonical schema for claims, the durable representation of every atom, and every operation that mutates the canonical record.

In one sentence: **Atoms knows what an atom is, where it lives, how it was born, how it changes, and how it relates to other atoms.**

## What it owns

| Concern | Notes |
|---|---|
| Atom schema | The shape of every atom and every subtype (`DecisionAtom`, `DefinitionAtom`, etc.). Field definitions, status enum, scope/premise structures. The schema is owned here because Extraction (Create) lives here — the producer of a record owns its shape. |
| Durable storage | Atoms persisted in a diffable form. (The local-MCP impl uses YAML in a git repo; other impls may differ. Persistence is an implementation choice; the conceptual contract is durability + diff-friendliness.) |
| Extraction (Create) | Fragments → proposed atoms. Pluggable per content-kind: decision extractors, definition extractors, dependency extractors, generic fallback. |
| Conflict detection | New atoms classified against canonical: `New` / `Duplicate` / `Reinforces` / `Refines` / `Supersedes` / `Contradicts` / `Compatible` / `Unclear`. Maintains an internal embedding-NN index for candidate finding; the index is private and not part of the public API. |
| Status transitions | **Present states** (atom remains in the snapshot): `Active` / `Hanging` / `Deferred` / `Deprecated`. **Removal outcomes** (atom no longer present; the intent is captured in the Change Log): `Superseded` / `Duplicate` / `Rejected` / `Removed`. Removal outcomes carry meta (superseding atom id, canonical-duplicate id, rejection rationale) so the why is recoverable without losing the intent in the snapshot diff. Implementation choice: an impl may persist a removal outcome as a tombstone status on the atom, or rely on snapshot history + Change Log entries to recover the intent — either is conformant. See [`11-flows.md`](./11-flows.md) for the state machine and [`05-orchestration.md`](./05-orchestration.md) for snapshot lifecycle. |
| Reference traversal | Primitive on `Atom.references` / `Atom.depends_on` — "atoms X points at" / "atoms pointing at X". The graph shape Atoms exposes externally. |

Conflict detection is part of Atoms because it operates entirely on atom state (status, supersession links, classification history). Its internal index exists to make conflict detection fast; consumers that want their own alias or subject indexes build them by walking the atom tree via the public read API.

### Status transitions triggered by Conflict

When Conflict classifies a new atom as `Supersedes` or `Refines` against an existing atom, two things happen in the new snapshot:

1. The existing atom is changed — deleted for `Supersedes`, content-updated for `Refines`.
2. Atoms walks the direct references *to* that changed atom (via the reference-traversal primitive) and flips each referrer's status to `Hanging`.

Both steps are Atoms primitives — reference traversal and status updates are part of Atoms's public API and used here internally. The floor case of impact propagation (atoms directly referencing a changed atom) is handled by Atoms alone, no optional capability required. [Blast Radius](./07-blast-radius.md), when wired up, extends the impact set beyond direct refs via premise-tag traversal and LLM-implicit analysis.

## API surface (conceptual)

The public read API is **tree and graph traversal** over atoms. Atoms does not expose by-field query APIs; callers that need indexed lookups build their own indexes by walking the tree.

**Read:**
- `get_by_id(atom_id)` — direct lookup by stable identifier.
- `list_scope(scope_path)` — atoms at one node of the atom tree.
- `list_children(scope_path)` — child scopes of a tree node.
- `traverse_references(atom_id, direction)` — walk the reference graph from an atom (incoming or outgoing).
- `changes_since(ref) → ordered_delta_stream` — the container's own change stream. Returns an ordered, typed sequence of events since `ref`: `Added(atom)`, `Updated(atom_id, fields)`, `StatusChanged(atom_id, from, to, rationale)`, `Deleted(atom_id)`. Consumed by [Hierarchical Navigation](./06-hierarchical-nav.md), [Projection](./04-projection.md), [Blast Radius](./07-blast-radius.md), [Quality](./10-quality.md) telemetry, and any other container that maintains derived state against the atom set. Order is significant; partial consumption is checkpointable via `ref`.

**Write (against the currently selected snapshot):**
- `propose(fragment, extractor_hints?) → ingest_id, summary` — runs extraction + conflict detection; writes the resulting atoms to the current snapshot; returns an identifier and stats summary (how many atoms added, conflict counts, unresolved questions).
- `update_status(atom_id, outcome, rationale, meta?)` — the single mutation API outside `propose`. `outcome` is one of the present-states (`Active` / `Hanging` / `Deferred` / `Deprecated`) or removal outcomes (`Superseded` / `Duplicate` / `Rejected` / `Removed`). Removal outcomes accept meta (e.g., `superseded_by`, `duplicate_of`) so the intent is captured. Internally routed through Status Manager.

Atoms operates against whichever snapshot is currently selected. **Snapshot selection** — which version Atoms reads from and writes to, when to switch, when to publish a snapshot as canonical — is owned by [Orchestration](./05-orchestration.md), not by Atoms. Concrete realizations of "snapshot" (a git working tree on disk, a database snapshot, a branch in a versioned store) are an implementation choice.

The interface contract — exact signatures, error model, idempotency rules — will be specified in `docs/interfaces/atoms.md`.

## Dependencies

- **[LLM Gateway](./09-llm-gateway.md)** *(non-optional infrastructure)* — for Extraction and Conflict's LLM judge.
- **[Ingestion](./02-ingestion.md)** *(non-optional)* — produces the normalized fragments that Extraction consumes.

No dependencies on optional containers (per the architecture principle).

## Components (preview of L3)

- **Schema** — atom + subtype definitions; validator.
- **Atom Store** — durable persistence (impl-specific).
- **Extraction Router** — picks extractors per content-kind.
- **Extractor[s]** — pluggable per content-kind (`DecisionExtractor`, `DefinitionExtractor`, `DependencyExtractor`, `GenericExtractor`, …).
- **Conflict Pipeline** — deterministic match → embedding NN → LLM judge.
- **Embedding Index** — internal to Conflict; not exposed externally.
- **Status Manager** — the single funnel for canonical atom mutation. Applies present-state transitions (`Active` / `Hanging` / `Deferred` / `Deprecated`) and removal outcomes (`Superseded` / `Duplicate` / `Rejected` / `Removed`, each with intent meta) to the Atom Store. The only component allowed to mutate canonical atom state.
- **Lookup APIs** — the read primitives (by-id, by-alias, by-reference, etc.).
- **Change Log** — maintains the ordered, append-only event log for this container. Receives notifications from Atom Store and Status Manager (`Added` / `Updated` / `StatusChanged` / `Deleted`). Serves `changes_since(ref)` and manages the ref / checkpoint surface that [Hierarchical Navigation](./06-hierarchical-nav.md), [Projection](./04-projection.md), [Blast Radius](./07-blast-radius.md), and [Quality](./10-quality.md) track against.

Full L3 detail lives in `docs/L3/03-atoms.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| Storage backend | YAML in git (local-MCP), Postgres + git-mirror (hybrid), object-store-backed (SaaS). |
| Extractor set | Minimal (decision + definition + generic) vs. full (+ dependency, capability, scenario, behavior, constraint). |
| Conflict pipeline depth | Deterministic-only (fast, low recall) → +NN (medium) → +LLM judge (highest fidelity). Implementations choose the cost/quality point. |
| Embedding model | Tied to the LLM Gateway's routing; per-impl trade-off between dim, latency, and recall. |
| Schema extensions | Tenants/orgs may add custom atom subtypes for their domain; the global schema defines the floor, not the ceiling. |
