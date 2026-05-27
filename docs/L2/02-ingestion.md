# Ingestion — Raw Inputs In

**Status: non-optional.** Without Ingestion, the system has nothing to extract from.

## Concern

Ingestion is the entry boundary for everything the system learns about. It accepts raw inputs from heterogeneous sources, normalizes them into a single shape, optionally filters out obvious noise, and persists the normalized fragment durably for provenance.

In one sentence: **Ingestion turns "something happened out there" into a normalized, addressable fragment that the rest of the system can reason about.**

## What it owns

| Concern | Notes |
|---|---|
| Source adapters | One per integration target (chat platform, version-control hosting, doc store, meeting platform, speech-to-text stream, manual file export, …). Each adapter handles auth, rate limits, format quirks for its source. |
| Normalization | Translate source-specific shapes into a single `NormalizedFragment` shape: stable `message_id`, sender, recipients, payload, timestamp, content kind, and a free-form `extras` map. |
| Relevance filtering | Cheap pre-extraction classifier that drops content unlikely to yield useful atoms. Tunable; biased toward false positives (keep marginal content; drop only the obvious). |
| Raw fragment store | Append-only archive of every accepted fragment with metadata. Immutable; never deleted in normal operation. Provenance audit depends on this. |

The raw store is conceptually distinct from the atom snapshots that [Atoms](./03-atoms.md) operates against. Raw fragments are append-only and not versioned — once ingested, they don't change. Multiple atom snapshots can reference the same raw fragments.

## API surface (conceptual)

**Write:**
- `accept(normalized_fragment) → fragment_id, accepted?` — runs the relevance filter, persists the fragment if accepted, returns its stable identifier plus whether it was kept. Idempotent on `message_id`.

**Read:**
- `get(fragment_id) → fragment` — fetch a fragment with its full metadata.
- `list(filter) → [metadata]` — iterate metadata by source / time range / sender / content kind. (Metadata only; fetch payloads explicitly via `get`.)
- `changes_since(ref)` — fragments added since a marker. The container's own change stream.

## Dependencies

- **[LLM Gateway](./09-llm-gateway.md)** — *only* if the relevance filter uses an LLM-based classifier; the simplest variant uses rules or a small local model and has no LLM Gateway dep.
- No dependencies on optional containers.

## Components (preview of L3)

- **Source Adapters** — pluggable, one per source. Each implements the same `accept` contract for the source's wire format and produces `NormalizedFragment`.
- **Normalizer** — the shared post-adapter step that validates and canonicalizes structure.
- **Relevance Filter** — pluggable classifier; can be a rule set, a small local model, or an LLM call.
- **Raw Store** — durable, append-only persistence layer (impl-specific backend).
- **Change Log** — maintains the ordered, append-only event log for this container. Receives `FragmentAccepted` / `FragmentRejected` notifications from Source Adapters / Relevance Filter. Serves `changes_since(ref)` for any downstream consumer that wants to react to new fragments (rarely used in steady-state, but useful for telemetry and re-runs).

Full L3 detail lives in `docs/L3/02-ingestion.md` (TBD).

## Variation points (where impls differ)

| Variation | Examples |
|---|---|
| Source adapter set | Minimal (manual file export only) → expanded (chat + version-control + doc store) → full (+ streaming STT for live capture). Each adapter is optional independently. |
| Normalization strictness | Strict schema-validated (rejects malformed) vs. permissive (accepts and tags problems for review). |
| Relevance filter | None (everything passes) → rules-only → small local classifier → LLM-based. Trade cost vs. recall. |
| Raw store backend | Filesystem + metadata SQLite (local-first); object store + relational DB (SaaS); committed-into-git (smallest setups). |
| Multi-tenancy | Single-tenant (one raw store per deployment); per-tenant isolation (prefixed buckets, per-tenant DB schemas). |

## Relationship to "context" of incoming fragments

A `NormalizedFragment` carries minimal source-specific structure: who sent it, who received it, when, what kind of content, the payload. Anything richer — thread structure, conversation context, document hierarchy — lives in the `extras` map and is interpreted by downstream extractors. Ingestion does not understand semantics; it preserves them.
