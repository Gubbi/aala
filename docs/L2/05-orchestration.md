# Orchestration — Lifecycle Coordinator

**Status: non-optional.** Every aala deployment includes some realization of Orchestration; it's the container that ties the capability set together and presents a single entry point to the outside world.

## Concern

Orchestration is the seam between the outside world and the capability containers inside aala. It owns three things that nothing else does:

1. **Snapshot lifecycle** — which version of atoms / projected prose / derived state the capability containers operate against, and how that selection changes over time.
2. **Tree registry** — the set of registered trees (bounded discourses such as `planning`, `implementation`) and their lifecycle (registration, rename, merge, split).
3. **Capability wiring and routing** — which capabilities are present in this deployment, and which one(s) should handle a given incoming request.

In one sentence: **Orchestration decides which snapshot is "now", which trees exist, and which capability handles a given request, then composes their calls into a single response.**

## The snapshot model

A **snapshot** is an addressable, coherent version of all snapshot-bound state. Atoms in a snapshot, projected prose in the same snapshot, and any derived indexes pinned to that snapshot together describe one consistent view of the world.

Snapshots have a small lifecycle:

| State | Meaning |
|---|---|
| **Working** | Mutable. Capabilities are allowed to write to it (e.g., `Atoms.propose`, and the projection facet re-deriving its host's prose). At most one snapshot is the "current working" snapshot at any moment. |
| **Canonical** | Immutable. Represents committed state that the system regards as the record of truth. Reads against canonical are stable. Multiple historical canonical snapshots may exist (the lineage of the record). |
| **Discarded** | Working snapshots that the user / system chose not to promote. Eligible for garbage collection. |

Snapshots have an identity (`snapshot_id`) and an optional parent (the snapshot they were forked from). The current working snapshot is what capabilities like Atoms (and the projection facet over their content) refer to when they say "the currently selected snapshot."

Snapshots are conceptual. Concrete realizations vary by implementation: a snapshot may be a git ref + working tree on disk, a row in a versioning table, an in-memory immutable data structure, a content-addressed manifest, or anything else with the right semantics. The capability containers don't know which — they ask Orchestration.

**Coherence**: at any moment, every capability container reading state operates against the *same* snapshot, so cross-container reads are coherent. Switching snapshots is an Orchestration operation, not a per-container one.

## The tree model

A **tree** is a bounded discourse identified by a `TreeId` (e.g., `planning`, `implementation`, `staging`). Trees are the unit at which same-claim atoms collapse — Duplicate detection runs within-tree; cross-tree similarity surfaces candidate equivalence edges instead.

Orchestration owns the tree registry and exposes lifecycle operations: register a new tree (which triggers Atoms to initialize the standard library for that tree), rename, merge, split. Tree migrations are coarse operations (per [`spec/12-edge-cases.md`](../../spec/12-edge-cases.md)) and are serialized against all other tree mutations.

## What it owns

| Concern | Notes |
|---|---|
| Snapshot manager | Creates, switches, publishes, discards snapshots. Knows the snapshot lineage. |
| Tree registry | Registers, lists, renames trees. Each tree has metadata (display name, description, atom count). |
| External API surface | The endpoint(s) outside callers (agents, CI jobs, webhooks, users) talk to. Realization depends on implementation (MCP tools, HTTP, CLI). |
| Routing | For an incoming request (an ingest, a read, a blast analysis), decide which capability container(s) handle it. Includes extractor routing by content kind, dispatch of reads (structured Atoms reads and projection-facet reads: `list` / `read` / `read_index`), tree assignment for ingest. |
| Lifecycle state | Tracks in-flight ingests, proposals, queries — the things that span multiple capability calls. |
| Capability registry | Knows which optional capabilities are wired in this deployment. Used to gracefully degrade requests when a capability is absent. |

## API surface (conceptual)

**Snapshot management:**
- `current_snapshot() → snapshot_ref`
- `fork(from?) → snapshot_ref`
- `switch_to(snapshot_id)`
- `publish_as_canonical(snapshot_id) → snapshot_ref`
- `discard(snapshot_id)`
- `list_snapshots(filter?) → [snapshot_ref]`

**Tree management:**
- `list_trees() → [tree_info]`
- `register_tree(tree, display_name, description?) → tree_info` — initializes standard-library classifications and predicate kinds for the tree.

**Request entry points:**
- `ingest(normalized_fragment, options?) → ingest_summary` — accepts a fragment; runs Ingestion, then Atoms.propose, then the affected hosts re-derive their projection facet and Blast Radius runs if wired and triggered; returns identifier + summary. `options.target_tree` overrides default tree assignment.
- `read(...)` — dispatches reads to the appropriate container: structured Atoms reads (`get_by_id`, `traverse_relations`, `list_by_classification`, …) and projection-facet reads (`list`, `read`, `read_index`, `changes_since`) over a host's own content. External agents compose answers over these read surfaces themselves (see [`docs/analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md)).
- `record_resolution(blast_id, atom_id, resolution, rationale, meta?) → blast_report` — forward reviewer resolution into Blast Radius.

**Capability discovery:**
- `capabilities() → [capability_info]`

## Dependencies

- **All non-optional capabilities** ([Ingestion](./02-ingestion.md), [Atoms](./03-atoms.md)) — Orchestration coordinates them on every entry-point call. The [projection facet](./04-projection.md) is internal to each contentful host (always Atoms); Orchestration dispatches reads to it but does not own it.
- **Optional capabilities** — used if wired up. Absence is graceful per the routing logic.
- **No direct LLM Gateway dependency** — Orchestration is pure coordination; LLM work happens inside the capability containers.

## Components (preview of L3)

- **Snapshot Manager** — creates, switches, publishes, discards. The single owner of snapshot identity and lineage.
- **Tree Registry** — registers, lists trees; triggers Atoms standard-library initialization on tree creation.
- **External API Surface** — the actual protocol surface (MCP tool handlers, HTTP routes, CLI commands).
- **Request Router** — content-kind → extractor mapping, read dispatch (structured Atoms reads and projection-facet reads), tree assignment. Configurable.
- **Lifecycle Tracker** — manages in-flight state for multi-step operations.
- **Capability Registry** — startup-time discovery; runtime "what's wired" queries.
- **Composer** — combines the outputs of multiple capabilities into a single response shape for the external caller.

Full L3 detail lives in [`docs/L3/05-orchestration.md`](../L3/05-orchestration.md).

## Variation points (where implementations differ)

| Variation | Examples |
|---|---|
| External API style | MCP server; HTTP + WebSocket; CLI subcommands; library API (embedded). |
| Snapshot backend | Git refs + working tree; versioning column in a relational DB; content-addressed manifest store; in-memory immutable structures. |
| Canonicalization model | aala-managed (publish_as_canonical mutates a head pointer); externally-managed (publish is a no-op or detection-only; canonical state is whatever the external store says). |
| Tree registry persistence | Alongside atoms in the snapshot; container-internal; external configuration. |
| Routing strategy | Static config-driven; capability-version-aware; dynamic / ML-routed. |
| Concurrency model | Single working snapshot (serialized writes); multiple working snapshots per user / branch / session (concurrent writes). |
| Lifecycle semantics | Fully synchronous (entry-point call returns when every downstream capability has completed); asynchronous with a polling API; event-streamed. |

## What Orchestration does NOT do

- It does not extract atoms — that's [Atoms](./03-atoms.md).
- It does not classify conflicts — that's [Atoms](./03-atoms.md).
- It does not render prose — that's the [projection facet](./04-projection.md) on each contentful container.
- It does not synthesize answers or generate documents — external agents compose over aala's read surfaces themselves (see [`docs/analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md)).
- It does not decide *what* canonical truth is — it decides *which snapshot is current* and *when promotion happens*; the content is owned by the capability containers.
