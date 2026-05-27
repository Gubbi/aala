# Orchestration — Lifecycle Coordinator

**Status: non-optional.** Every aala deployment includes some realization of Orchestration; it's the container that ties the capability set together and presents a single entry point to the outside world.

## Concern

Orchestration is the seam between the outside world and the capability containers inside aala. It owns two things that nothing else does:

1. **Snapshot lifecycle** — which version of atoms / projections / derived state the capability containers operate against, and how that selection changes over time.
2. **Capability wiring and routing** — which capabilities are present in this deployment, and which one(s) should handle a given incoming request.

In one sentence: **Orchestration decides which snapshot is "now" and which capability handles a given request, then composes their calls into a single response.**

## The snapshot model

A **snapshot** is an addressable, coherent version of all snapshot-bound state. Atoms in a snapshot, projections in the same snapshot, and any derived indexes pinned to that snapshot together describe one consistent view of the world.

Snapshots have a small lifecycle:

| State | Meaning |
|---|---|
| **Working** | Mutable. Capabilities are allowed to write to it (e.g., `Atoms.propose`, `Projection.re_render`). At most one snapshot is the "current working" snapshot at any moment. |
| **Canonical** | Immutable. Represents committed state that the system regards as the record of truth. Reads against canonical are stable. Multiple historical canonical snapshots may exist (the lineage of the record). |
| **Discarded** | Working snapshots that the user / system chose not to promote. Eligible for garbage collection. |

Snapshots have an identity (`snapshot_id`) and an optional parent (the snapshot they were forked from). The current working snapshot is what capabilities like Atoms and Projection refer to when they say "the currently selected snapshot."

**Key property: snapshots are conceptual.** Concrete realizations vary by implementation: a snapshot may be a git ref + working tree on disk, a row in a versioning table, an in-memory immutable data structure, a content-addressed manifest, or anything else with the right semantics. The capability containers don't know which — they ask Orchestration.

**Coherence**: at any moment, every capability container reading state operates against the *same* snapshot, so cross-container reads are coherent. Switching snapshots is an Orchestration operation, not a per-container one.

## What it owns

| Concern | Notes |
|---|---|
| Snapshot manager | Creates, switches, publishes, discards snapshots. Knows the snapshot lineage. |
| External API surface | The endpoint(s) outside callers (agents, CI jobs, webhooks, users) talk to. Realization depends on impl (MCP tools, HTTP, CLI). |
| Routing | For an incoming request (an ingest, a query, a render trigger), decide which capability container(s) handle it. Includes extractor routing by content kind, intent routing for queries. |
| Lifecycle state | Tracks in-flight ingests, proposals, queries — the things that span multiple capability calls. |
| Capability registry | Knows which optional capabilities are wired in this deployment. Used to gracefully degrade requests when a capability is absent (e.g., Synthesis falls back from Hierarchical Navigation to raw Atom traversal). |

## API surface (conceptual)

**Snapshot management:**
- `current_snapshot() → snapshot_id` — what every capability is currently operating against.
- `fork(from?) → snapshot_id` — create a new working snapshot (defaults to forking from canonical).
- `switch_to(snapshot_id)` — make the given snapshot the current working one.
- `publish_as_canonical(snapshot_id) → canonical_ref` — promote a working snapshot to canonical. May be a no-op in deployments where canonicalization happens outside aala (see variation points).
- `discard(snapshot_id)`
- `list_snapshots() → [snapshot_meta]`

**Request entry points:**
- `ingest(normalized_fragment) → ingest_id, summary` — accepts a fragment; runs Ingestion, then Atoms.propose, then any non-optional follow-on capabilities (Projection re-render of affected sections, Blast Radius for decision-type atoms if wired); returns identifier + summary.
- `query(intent, hints?) → response` — routes to Synthesis if present; falls back to raw Atom traversal if not. Returns response with provenance.
- `re_render(scope?) → render_summary` — explicit projection re-render trigger.

**Capability discovery:**
- `capabilities() → [capability_name, version]` — list which optional capabilities are wired up. Lets clients (especially Synthesis) reason about what's available.

## Dependencies

- **All non-optional capabilities** ([Ingestion](./02-ingestion.md), [Atoms](./03-atoms.md), [Projection](./04-projection.md)) — Orchestration coordinates them on every entry-point call.
- **Optional capabilities** — used if wired up. Absence is graceful per the routing logic.
- **No direct LLM Gateway dependency** — Orchestration is pure coordination; LLM work happens inside the capability containers.

## Components (preview of L3)

- **Snapshot Manager** — creates, switches, publishes, discards. The single owner of snapshot identity and lineage.
- **External API Surface** — the actual protocol surface (MCP tool handlers, HTTP routes, CLI commands).
- **Request Router** — content-kind → extractor mapping, intent → capability mapping. Configurable.
- **Lifecycle Tracker** — manages in-flight state for multi-step operations (e.g., an ingest that spans Ingestion + Atoms + Projection).
- **Capability Registry** — startup-time discovery; runtime "what's wired" queries.
- **Composer** — combines the outputs of multiple capabilities into a single response shape for the external caller (e.g., an ingest response that includes Ingestion's fragment_id, Atoms's classification stats, Projection's affected sections, Blast Radius's report).

Full L3 detail lives in `docs/L3/05-orchestration.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| External API style | MCP server (local-MCP) ; HTTP + WebSocket (SaaS) ; CLI subcommands (batch / CI use) ; library API (embedded). |
| Snapshot backend | Git refs + working tree (local-MCP); versioning column in a relational DB (SaaS); content-addressed manifest store; in-memory immutable structures (tests). |
| Canonicalization model | aala-managed (publish_as_canonical is a real operation that mutates a head pointer) ; externally-managed (publish is a no-op or detection-only; canonical state is whatever the external store says; aala observes and updates its view). |
| Routing strategy | Static config-driven; capability-version-aware; dynamic / ML-routed. |
| Concurrency model | Single working snapshot (serialized writes); multiple working snapshots per user / branch / session (concurrent writes). |
| Lifecycle semantics | Fully synchronous (entry-point call returns when every downstream capability has completed); asynchronous with a polling API; event-streamed. |

## What Orchestration does NOT do

- It does not extract atoms — that's [Atoms](./03-atoms.md).
- It does not classify conflicts — that's [Atoms](./03-atoms.md) (Conflict is a primitive of Atoms).
- It does not render prose — that's [Projection](./04-projection.md).
- It does not synthesize answers — that's [Synthesis](./08-synthesis.md).
- It does not decide *what* canonical truth is — it decides *which snapshot is current* and *when promotion happens*; the content is owned by the capability containers.
