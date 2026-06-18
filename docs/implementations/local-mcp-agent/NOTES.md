# local-mcp-agent — Design Notes

**Status: discussion notes, not formal documentation.** Captures the design conversation around the first reference implementation of aala. Will be turned into proper docs (`L3.md`, `config.md`, `setup.md`, etc.) when the implementation is built. Until then, this file is the source of truth for what we agreed on.

## Premise

This is one specific implementation of the aala spec. It targets the simplest interesting use case: a single developer (or a small team) managing a documentation repo on their own machine, with an AI agent (Claude Code or similar) as the only client. No SaaS, no multi-tenancy, no live capture.

## Process model

- **aala is a single OS process** installed via `brew install` (or equivalent package manager).
- The process is a **stdio MCP server** that the AI agent launches as a subprocess.
- One process per agent session. Long-running daemon mode is out of scope for v1; if spin-up time becomes a problem we add a daemon later.

## Where state lives

| Location | Contents | Lifetime |
|---|---|---|
| `~/.aala/config.yaml` | Global aala config (LLM gateway tool choice, default routing) | Per-user |
| `~/.aala/cache/<repo-id>/stats.sqlite` | Stats derived from the repo working tree | Cache; rebuildable from working tree |
| `~/.aala/cache/<repo-id>/embeddings.sqlite` | Conflict-NN embeddings | Cache; rebuildable |
| `~/.aala/cache/<repo-id>/<other indexes>.sqlite` | Other derived indexes (tree indexes, claim-to-section, etc.) | Cache; rebuildable |
| `<user-repo>/.aala/config.yaml` | Per-repo aala config (extractor set, scope overrides, etc.) | Committed to user's git |
| `<user-repo>/claims/` | Atom YAMLs (the canonical store) | Committed |
| `<user-repo>/projections/` | Generated prose | Committed |
| `<user-repo>/raw/` | Mirrored raw inputs | Committed |

Notable: **no aala runtime state lives inside the user's repo**. All derived indexes / caches / session data are in `~/.aala/`. This means branch checkouts, stashes, `git pull`s don't churn aala's state.

## Snapshot model

- A snapshot in this impl IS the git working tree.
- "Current snapshot" = whatever the working tree currently shows.
- "Canonical" = whatever the user has committed (last commit on the current branch is canonical from aala's view).
- Switching snapshots = `git checkout` (handled by the user, observed by aala on next call).
- Publishing as canonical = `git commit` (handled by the user, observed by aala on next call).

**aala does NOT manage branching, commits, or PRs.** It writes uncommitted files to the working tree and lets the user own the git lifecycle.

## Freshness handling

- v1: aala is **stateless w.r.t. the repo on startup**. On each MCP invocation, it checks whether its cache matches the working tree (by comparing a single content hash) and rebuilds if stale.
- v2 optimization (when needed): per-branch cache keyed on last-reconciled commit-id; check if any raw inputs exist between the cached commit and current HEAD; fast-forward if yes, stay as-is if not.

The principle: working tree is the source of truth; everything in `~/.aala/cache/` is a derived index that can be rebuilt at any time.

## The two-step ingestion flow

1. **External AI agent** (Claude Code, etc.) reads raw input files in their original formats (Slack export, meeting transcript, doc, etc.).
2. The agent **normalizes** each fragment to the standard `NormalizedFragment` shape (message_id, sender, recipients, payload, timestamp, content_kind, extras).
3. The agent calls the **MCP server's ingest tool** with the normalized content.
4. The MCP server routes through Ingestion → Atoms → (Blast Radius if a decision atom is identified). Atoms' projection facet re-renders the affected canonical prose as part of the write.
5. Returns stats: atoms added, conflict counts, unresolved questions, blast reports if any.

The agent does the per-source format adapting (which would otherwise be Ingestion's pluggable Source Adapters in the global L2). In this impl, the Source Adapter is "an external AI agent following the NormalizedFragment contract."

## The agent IS the UI

- No web UI, no HTTP server, no custom review surface.
- The agent communicates findings (atom additions, blast radius reports, unresolved questions) to the user in chat.
- The agent uses its native diff tool (the same one used for code review) to show projection diffs.
- User reviews via `git diff` in their editor or via the agent's diff display.
- User approves by `git add` + `git commit`. aala observes the commit on next call.

This sidesteps building a Live Participant UI / Review UI entirely. Real-time streaming concerns are explicitly parked.

## MCP write side

- The MCP exposes an `ingest` tool that writes uncommitted YAMLs and markdown to the working tree.
- It does NOT manage branches, stashes, or commits.
- After `ingest`, the user reviews the working tree changes through their existing tooling.

## MCP read side

- Each `ingest` call returns **stats** indicating what changed (counts + identifiers).
- The MCP also exposes **sliced read APIs**: get unresolved questions in a scope, fetch a blast report, list atoms by scope, etc.
- The agent uses these to have an incremental discussion with the user rather than dumping everything in one shot.
- The agent can also `Read` the actual files in the working tree directly (since they're just YAML/markdown).

## Capabilities wired in v1

| Container | Wired? | Notes |
|---|---|---|
| Ingestion | yes | Single "agent-driven" source adapter. No webhooks, no STT. |
| Atoms | yes | Storage = YAML files in git. Conflict pipeline: deterministic + embedding NN + LLM judge. Embedding index in SQLite. Implements the projection facet over its claims (template + cached LLM narrative; cache in SQLite) — canonical claim prose, glossary, and index. |
| Orchestration | yes | MCP server is the External API Surface. Snapshot Manager = "current working tree" detector. |
| LLM Gateway | yes | Wraps LiteLLM (probable choice; deployer configures models). |
| Hierarchical Navigation | likely yes | Trees stored as JSON sidecar files in the cache dir. by-domain + by-team + by-glossary axes. |
| Blast Radius | yes | Direct refs + premise tags + LLM-implicit. Important for the use case. |
| Quality | not in v1 | Golden sets and benchmarks deferred. Add when we have enough usage to want to measure. |

## LLM Gateway choice

- **Underlying tool**: LiteLLM (most likely). Wraps everything behind one API; supports cloud + local; deployer maps models to use-case keys in their LiteLLM config.
- **Models**: deployer's choice. Configured in `~/.aala/config.yaml` mapping each registered use-case key to a model identifier.
- **Suggested defaults** (for the brew-installed package to bootstrap with):
  - `aala.extraction` → top-tier cloud (whatever the deployer has)
  - `aala.conflict_judge` → top-tier cloud
  - `aala.blast_implicit` → top-tier cloud (cost driver but high value)
  - `aala.projection_narrative` → mid-tier cloud, cached
  - `aala.label_synthesis` → mid-tier cloud
  - `aala.faithfulness_judge` → top-tier cloud
  - `aala.relevance_filter` → small local (or cheap cloud)
  - `aala.embedding_default` → cloud embedding model
- Privacy-sensitive deployers can override any of these to local (Ollama / vLLM) via LiteLLM config.

## What the agent contract looks like

The MCP server exposes a small set of tools. Sketch:

```
aala.ingest(normalized_fragment) → ingest_summary
aala.get_unresolved(scope?) → [unresolved_item]
aala.get_blast_report(blast_id?) → blast_report
aala.record_resolution(blast_id, atom_id, resolution, rationale) → updated_report
aala.list(prefix?) → [path]                      # projection facet: enumerate prose
aala.read(path) → { content, provenance }        # projection facet: fetch prose + provenance
aala.read_index(root?) → page_index              # projection facet: navigable index tree
aala.refresh() → cache_status   # explicit re-baseline; usually not needed
aala.capabilities() → [capability]
aala.stats(scope?) → stats_summary
```

The agent calls these. The agent decides what to show the user when — pagination, drill-downs, summaries.

Q&A and document generation (ADRs, pitches, comparisons, walkthroughs) are NOT aala tools. The **local agent itself** answers questions and composes documents by consuming aala's read surfaces — Atoms' structured reads (graph queries by scope / classification / relation) plus the projection facet (`read_index` to navigate, `read` to fetch canonical prose with provenance) — and composing the answer with its own LLM. See [`docs/analysis/agent-integration-pattern.md`](../../analysis/agent-integration-pattern.md).

## Deviations from the spec (and where we lean on optional contracts)

- **Canonicalization**: this impl is externally-managed. `Orchestration.publish_as_canonical` is detection-only; the user commits via git and aala observes.
- **Source adapter set**: minimal. Just the agent-driven one. Standard Source Adapters (Slack webhook, etc.) are not implemented.
- **Quality**: omitted in v1. The capability declaration shows `quality: { optional: true, wired: false }`.
- **Streaming projection**: not implemented. Projection is batch-only in v1.
- **Storage backend**: git working tree for snapshot-bound state; SQLite in `~/.aala/cache/<repo-id>/` for derived indexes.
- **Concurrency model**: single working snapshot, serialized writes (only one ingest at a time, since this is single-user single-agent).

All deviations are within what the spec permits — the impl declares its wired capabilities via `Orchestration.capabilities()` and clients can adapt.

## Open questions / things to figure out when building

- **Repo identification**: how does aala identify "which repo" it's managing on a given call? Likely from the working directory the MCP server is launched in, but needs a clean rule.
- **Embedding cache invalidation**: when an atom is modified, its embedding must be regenerated. Triggered by Atoms's `Updated` event on its delta stream.
- **Working tree drift detection**: comparing a single content hash on each MCP call — fast in most cases, but what about huge repos? May need to memoize.
- **MCP tool naming**: should match conventions of the MCP ecosystem (`aala.<verb>`).
- **Stats schema**: what's in a `stats_summary`? Counts (atoms by status, blast reports by state, unresolved questions). Needs a defined shape.
- **Concurrent MCP calls**: in principle the agent issues one tool call at a time, but if it ever issues two in parallel, the impl needs to serialize correctly.
- **Configuration validation**: at startup, validate that the deployer's `~/.aala/config.yaml` has every registered use-case key mapped to a model. Refuse to start otherwise.
- **First-run experience**: what happens the first time aala is invoked on a fresh repo? Probably: scan the working tree, build initial cache, no atoms exist yet.
- **Existing-atoms-in-repo recovery**: what if a user has hand-edited `claims/*.yaml` files? Need to handle this gracefully — they're valid atoms, just not produced by our extraction.
- **Branch tracking**: when the user switches branches, the working tree changes. v1's "rebuild on next call" approach should handle this, but worth testing.

## What this impl proves

This impl is the minimum viable aala — proving the architecture works for the simplest real use case. Specifically:

- One process. One user. One repo.
- No custom UI. The agent renders.
- All snapshot-bound state in git; all derived state in a per-user cache dir.
- Pluggable LLM gateway via LiteLLM.

If it works here, the SaaS multi-tenant impl ("cloud-aala") becomes a matter of swapping concrete realizations: snapshot backing → versioning DB, single-tenant config → per-tenant routing, agent → web UI, etc.

## Next steps when we come back to this

1. Pick a language for the MCP server (Rust seems natural — fast startup, single static binary, good MCP libraries; alternative is TypeScript).
2. Pick the persistence libraries (SQLite for caches, libgit2 or git CLI for git ops).
3. Write the L3 layout for this impl specifically — which concrete components, which libraries, how the MCP tool calls map to internal calls.
4. Write the `setup.md` for the brew-install user experience.
5. Write the `config.md` for the deployer-facing config surface.
6. Build it.
