# 13 — Wire Profiles

This chapter binds the wire formats, transports, and storage media through which conformant implementations exchange bytes.

## Definition

A **wire profile** is a tuple `(serialization, transport or medium, encoding rules)` that fully specifies how the data model and interface signatures land on a network or a storage medium.

Two implementations are wire-interoperable **only if** they share at least one supported wire profile. Sharing a profile is necessary, never sufficient — the full conditions (shared profile, encoding rules honored, identifier formats verbatim, semantic conformance) are defined once in [02 — Conformance § Cross-implementation interoperability](./02-conformance.md#cross-implementation-interoperability).

### Profile classes

Every profile in the registry declares one **class**:

- **`rpc`** — carries the **call surface**: method bindings, session establishment, streaming, and the error envelope. The interface contracts in [`interfaces/`](../interfaces/) are discharged over `rpc`-class profiles.
- **`persistence`** — binds the **at-rest representation** of snapshot state on a storage medium, so that independently built implementations can share one store or exchange snapshots. A persistence-class profile is not a call surface: it defines no method bindings, no in-band sessions, and no error envelope — errors are runtime artifacts of calls ([10 — Error Model](./10-error-model.md)), and calls ride an `rpc`-class profile.

A conformant implementation supports at least one `rpc`-class profile and declares persistence-class profiles only in addition, never alone — the conformance statement is [02 § Wire profile declaration](./02-conformance.md#wire-profile-declaration). Call-level interop requires a shared `rpc`-class profile; implementations sharing only a persistence-class profile interoperate at the store level (each can read and continue the other's store) but not at the call level.

## Required interoperability claims

The implementation-side obligations are bound in [02 — Conformance](./02-conformance.md): at least one `rpc`-class profile supported, every supported profile declared in `Orchestration.capabilities().wire_profiles` ([02 § Wire profile declaration](./02-conformance.md#wire-profile-declaration)), every interface contract honored within each declared `rpc`-class profile, and every declared profile's encoding rules honored ([02 § Interface conformance](./02-conformance.md#interface-conformance)). An implementation MAY support multiple profiles concurrently (e.g., HTTP for external callers, MCP for local agents, git for snapshot persistence).

The caller-side selection rule is in [§ Profile declaration](#profile-declaration).

## Profile-independent encoding

Identifier formats (`AtomId`, `FragmentId`, `SnapshotId`, `Ref`, `IngestId`, `RequestId`, `SystemOpId`, `BlastId`, `TreeId`) are normative per [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md) and MUST be preserved verbatim across every wire profile. The `correlation_id` on change events is one of these (an `IngestId`, `RequestId`, or `SystemOpId`) and is likewise preserved verbatim. No profile-specific escaping or transformation is permitted for identifiers — the character set (`[A-Za-z0-9_\-.:]`) is safe for JSON strings, YAML scalars, URL path segments, and MCP tool arguments.

The identifier character set is **not** universally filesystem-safe: `:` is reserved in Windows filenames and historically problematic on other filesystems. No profile in this registry uses an identifier as a file or directory name; a profile that derives filesystem names binds its own derivation rule ([`aala-wire/yaml+git` § Repository layout](#repository-layout)).

For paths (`SectionPath`, `ScopePath`), profiles MAY apply standard escaping (URL encoding for HTTP, etc.) at the wire boundary, but the underlying string remains stable through round trips.

## Wire profile registry

The following profiles are part of the `aala-v0.1` published contract.

| Profile | Class | Serialization | Transport / medium |
|---|---|---|---|
| [`aala-wire/json+http`](#aala-wirejsonhttp) | `rpc` | JSON | HTTP |
| [`aala-wire/json+mcp`](#aala-wirejsonmcp) | `rpc` | JSON | Model Context Protocol |
| [`aala-wire/yaml+git`](#aala-wireyamlgit) | `persistence` | YAML 1.2 | Git repository |
| [`aala-wire/protobuf+grpc`](#aala-wireprotobufgrpc-reserved--not-yet-specified) | `rpc` (RESERVED) | — | — |

### `aala-wire/json+http`

| Aspect | Specification |
|---|---|
| Class | `rpc` |
| Serialization | JSON (RFC 8259) |
| Transport | HTTP/2 with TLS in production; HTTP/1.1 acceptable for local-only deployments |
| Method binding | Each interface method maps to `POST <base-url>/<container>/<method>` with the call arguments as the JSON request body and the method's return value as a single JSON document in the response body. This is a deliberate RPC binding: methods do not map onto REST resource semantics, and because reads also ride `POST` (an unsafe method under RFC 9110), HTTP-level caching does not apply |
| Status codes | `200 OK` on every successful call (including replays recognized under an idempotency contract). On error, the response body is the error envelope and the status code follows [§ HTTP status codes](#http-status-codes). The envelope is authoritative: callers branch on its classification ([10 § Callers branch on classification](./10-error-model.md#callers-branch-on-classification-not-on-names)), never on the status code, which exists to route generic HTTP machinery (proxies, retry middleware) |
| Identifier in URL | Identifiers MAY appear in URL paths verbatim (the character set is URL-safe by spec); no escaping required |
| Idempotency | No transport-level key channel. Idempotency contracts are payload-derived per the registry in [shared-types § Idempotency](../interfaces/00-shared-types.md#idempotency) — the request body already carries every declared key |
| Streaming methods | `chat_streaming` is the only streamed method: the response is Server-Sent Events (`Content-Type: text/event-stream`); each `ChatChunk` is one SSE event whose `data` is the chunk's JSON; a stream-terminating failure is a final SSE event with `event: error` whose `data` is the error envelope. Every other method — including `changes_since` — is plain request/response |
| Field naming | `snake_case` for all field names, matching the TypeScript interface declarations |
| Atom IDs | Plain JSON strings; format per `00-shared-types.md` |
| Datetimes | RFC 3339 `date-time` strings with explicit UTC offset (no implicit UTC), per [shared-types § Timestamps and durations](../interfaces/00-shared-types.md#timestamps-and-durations) |
| Durations | ISO 8601 duration strings (e.g., `"PT30S"`) — durations have no RFC 3339 profile |
| URIs | Plain JSON strings, validated against RFC 3986 |
| Numbers | Standard JSON number type; no precision-sensitive values (use string encoding for precise decimals) |
| Booleans | JSON `true` / `false` |
| Null / absence | Omit optional fields rather than serialize `null` (smaller payloads + clearer intent) |
| Error envelope | `{ "error": { "tier": "activity", "activity": "...", "end_state": "...", "message": "...", "fix_domain": "...", "recoverability": "...", "severity": "...", "trace_id": "...", "cause": { "tier": "primitive", "type": "...", ... } } }` — the serialized `ActivityError` shape from [`00-shared-types.md`](../interfaces/00-shared-types.md#errors--the-three-tier-model), carrying the full three-tier chain (see [10 — Error Model](./10-error-model.md)). An Operation Error wraps an Activity Error under `cause` when produced at a touch point |
| Correlation | The envelope's `trace_id` — the operation's `correlation_id` ([10 § `trace_id`](./10-error-model.md#trace_id--the-operations-correlation_id-carried-on-the-error)) — MUST also be returned in the `Aala-Correlation-Id` response header on every error response, and MAY be set on success responses. It is not a W3C Trace Context trace-id: the `traceparent` / `tracestate` headers, where used, carry standard W3C-formatted values per [10 § Mandated observability](./10-error-model.md#mandated-observability), and an implementation MUST NOT place the `correlation_id` — or anything derived from it — in them. Implementations MAY additionally propagate the correlation id as the `aala.correlation_id` entry in W3C Baggage |
| Session establishment & access | A session is established by a Bearer token in the `Authorization` header; the deployment resolves the token to the session's grant set and initial snapshot binding ([02 — Conformance § Access control](./02-conformance.md#access-control)). Grant-check failures surface as the `authorizing` pairs in the standard error envelope. Token format and identity provider are deployment-defined |
| Versioning | Container version in the `Aala-Version` request and response header (unprefixed per RFC 6648); mismatch handling per [11 — Versioning](./11-versioning.md) |
| Delta-Stream paging | Each `changes_since` call returns one `ChangesPage` as a single JSON document. Caught-up is signaled by `next_ref` being **absent** — never `"next_ref": null` (encoding rule 2). The caller re-invokes with the returned `next_ref` until it is absent ([05 § Paging](./05-delta-streams.md#paging)) |

#### HTTP status codes

The status code of an error response derives from the error's `(activity, end_state)` pair and its registered classification ([Appendix A § Activity end-states](./appendix-a-registries.md#activity-end-states)) — first matching row wins:

| Error (first match wins) | Status |
|---|---|
| `authorizing` / `unauthenticated` | `401 Unauthorized` |
| `authorizing` / `denied` | `403 Forbidden` |
| `end_state` = `not_found` | `404 Not Found` |
| `end_state` = `non_fast_forward` | `409 Conflict` |
| `end_state` = `throttled` | `429 Too Many Requests` |
| registered `recoverability` = `retryable` | `503 Service Unavailable` |
| registered `fix_domain` = `client` | `400 Bad Request` |
| everything else | `500 Internal Server Error` |

When the envelope carries `retry_after_ms`, the response SHOULD set the `Retry-After` header (in seconds, rounded up). Because the table keys on registered classification, `(activity, end_state)` pairs added by future spec versions inherit a status code without a profile change.

### `aala-wire/json+mcp`

| Aspect | Specification |
|---|---|
| Class | `rpc` |
| Serialization | JSON (as MCP message content) |
| Transport | Model Context Protocol over its standard transports — stdio or Streamable HTTP — at protocol revision `2025-03-26` or later (the revision that defines the Streamable HTTP transport) |
| Method binding | Each interface method maps to an MCP tool (`<container>.<method>`) with the call arguments as the tool's `arguments` and the return value as the tool result content (a single JSON document) |
| Idempotency | Same as `aala-wire/json+http`: payload-derived per [shared-types § Idempotency](../interfaces/00-shared-types.md#idempotency); no transport-level key channel |
| Streaming methods | MCP tool calls return complete results; the protocol does not stream partial tool results. `chat_streaming` degrades to complete delivery — its tool result carries the final `ChatResponse` (the terminal `ChatChunk`); implementations MAY report interim progress via MCP progress notifications. `changes_since` pages via repeated tool calls |
| Field naming | `snake_case` |
| Atom IDs / Datetimes / Durations / URIs / Numbers / Booleans / Null | Same as `aala-wire/json+http` |
| Error envelope | A failed method call returns an MCP tool **execution error** (tool result with `isError: true`) whose content is the serialized error envelope — the same `ActivityError` JSON as `aala-wire/json+http`, `trace_id` included. JSON-RPC protocol errors (malformed request, unknown tool) are transport-level and are not aala Activity Errors |
| Session establishment & access | The MCP connection is the session: establishment context is taken at connection initialization per the transport's own authorization model (stdio: the process owner; remote transports: the MCP authorization specification) and resolves to the session's grant set and initial snapshot binding ([02 — Conformance § Access control](./02-conformance.md#access-control)) |
| Versioning | Declared in the MCP server's metadata at initialization (the `initialize` exchange) |
| Delta-Stream paging | Tool call returning one `ChangesPage`; the caller re-invokes with `next_ref` until it is absent |

### `aala-wire/yaml+git`

A **persistence-class** profile: it binds the at-rest representation of a deployment's snapshot space on a git repository, so that independently built implementations can share one store or exchange snapshots, and so that snapshot review rides ordinary git tooling (diffs, PRs). It is not a call surface.

| Aspect | Specification |
|---|---|
| Class | `persistence` — binds at-rest state, not calls. Interface methods, sessions, and the error envelope ride an `rpc`-class profile; declaring only this profile does not satisfy conformance ([02 § Wire profile declaration](./02-conformance.md#wire-profile-declaration)) |
| Serialization | YAML 1.2, **core schema**. Implementations MUST parse and emit in YAML 1.2 mode |
| Medium | A git repository. The snapshot lifecycle ([04 — Snapshots](./04-snapshots.md)) maps onto git: a working snapshot is a branch (fork = branch creation from the parent's commit); each committed operation is one git commit on the snapshot's branch ([§ Event journal](#event-journal)); the canonical pointer is a designated branch head, and publish is a fast-forward update of that branch — the compare-and-swap of [04 § Canonicalization](./04-snapshots.md#canonicalization), with git's fast-forward rule realizing `non_fast_forward`; discard deletes the snapshot's branch |
| Repository layout | Normative, in [§ Repository layout](#repository-layout). Identifiers never appear as file or directory names |
| Field naming | `snake_case` |
| Atom IDs | YAML scalars (plain strings); format per `00-shared-types.md` |
| Datetimes / Durations / URIs | Same as `aala-wire/json+http`, in YAML scalar form |
| Numbers | YAML number types; precise decimals as quoted strings |
| Booleans | YAML `true` / `false` only. Under YAML 1.1 resolution rules `yes` / `no` are booleans; under the YAML 1.2 core schema they are plain strings — emitting them invites cross-parser divergence, so implementations MUST NOT emit `yes` / `no` for boolean values |
| Null / absence | Omit fields entirely rather than serialize `null` or `~` |
| Error envelope | None, by construction. Errors are runtime artifacts of calls, and this profile carries no calls — nothing error-shaped exists at rest. The error-model obligations ([02 § Error-model conformance](./02-conformance.md#error-model-conformance-the-cross-vendor-contract)) are discharged on the implementation's `rpc`-class profile |
| Access | Realized by the medium's own access control — git-remote credentials (SSH key / HTTPS token) and repository / filesystem permissions; the deployment maps those permissions onto the grant model ([02 — Conformance § Access control](./02-conformance.md#access-control)). No in-band session context exists |
| Versioning | Declared in the top-level `metadata/version.yaml` file |
| Event journal | Normative, in [§ Event journal](#event-journal) |

#### Repository layout

The layout is normative — two implementations declaring this profile MUST be able to read and continue each other's repositories:

```
metadata/version.yaml      # contract version + profile metadata
metadata/trees.yaml        # registered trees: a list of { tree, display_name, description? }
atoms/<tree-dir>/*.yaml    # the tree's atoms, grouped multiple-atoms-per-file
```

- **Tree directories.** `<tree-dir>` is the `<opaque>` portion of the `TreeId` (the `tree:` prefix is constant, so the mapping is reversible). The opaque character set (`[A-Za-z0-9_\-.]+`) is portable across filesystems, including Windows.
- **Atom files.** Every `.yaml` file under a tree directory carries a top-level `atoms:` list. The tree's atom set is the union of those lists; a given atom `id` MUST appear in exactly one file. Readers enumerate files — they never construct filenames.
- **Grouping.** Files hold **multiple atoms per file**; one-file-per-atom is not a conformant layout for this profile (it scales poorly in git and on filesystems, and identifiers MUST NOT be used as filenames — the character set contains `:`, reserved on Windows). File names below a tree directory are implementation-chosen but MUST use only characters in `[A-Za-z0-9_\-.]`.
- **Determinism.** Within a file, atoms MUST be sorted by `id` (bytewise ascending), and a write MUST keep an unchanged atom in its current file — so diffs across commits, and across implementations sharing the store, stay reviewable.
- **Tombstones.** Atoms in removal-outcome states are ordinary entries (the state is in their `status`) and remain in place until garbage collection purges them ([07 — Atom Lifecycle](./07-atom-lifecycle.md)).
- **Extensions.** An implementation MAY persist additional state (ingestion records, indexes, …) in directories outside `metadata/` and `atoms/`; such state is implementation-defined and outside this profile's interop.

#### Event journal

Each committed operation is **one git commit** on the snapshot's branch (the operation-grain atomicity of [04 § Operation isolation and concurrency](./04-snapshots.md#operation-isolation-and-concurrency)), and the commit message carries the operation's events:

- **Subject line:** free-form summary.
- **Body** (after the blank line): one YAML 1.2 document with two top-level keys — `correlation_id:` (the operation's correlation id) and `events:` (the sequence of Atoms-container `ChangeEvent`s the operation emitted, in emission order, serialized under this profile's encoding rules).

This makes the journal comparable across implementations sharing a repository:

- A Delta-Stream `Ref` minted off this store carries the commit id in its `<opaque>` portion (`ref:<commit-id>`).
- `Atoms.changes_since(ref)`, served by an implementation over an `rpc`-class profile, is realized as `git log --reverse <ref-commit>..<branch-head>` — a commit-range selection — concatenating each commit's `events:` list in commit order and preserving each list's internal order. (`git log --since` filters by date and cannot express this range.)
- `ref = null` — the start of the working snapshot's own stream ([05 — Delta Streams](./05-delta-streams.md)) — corresponds to the range from the branch's fork point: `<fork-commit>..<branch-head>`.
- The journal binds the **Atoms** Delta Stream only. Other containers' streams (Ingestion, Projection, …) are runtime artifacts served over the `rpc`-class profile.

### `aala-wire/protobuf+grpc` (RESERVED — not yet specified)

Listed for forward compatibility. An implementation that supports gRPC may declare this `rpc`-class profile when its full specification lands (anticipated v0.2 or later).

## Common encoding rules

These rules govern document emission in every profile — JSON and YAML alike:

1. **Field ordering** — implementations SHOULD emit fields in the declaration order of the TypeScript interface, as a readability preference. Serialized objects are unordered (RFC 8259 for JSON; YAML mappings likewise); consumers MUST NOT depend on member order.
2. **Optional fields** — omit when absent. Do NOT emit `"field": null`.
3. **Empty collections** — emit `"field": []` or `"field": {}` rather than omit (so consumers can distinguish "no items" from "field not present").
4. **Discriminated unions** — the `kind` field SHOULD be emitted first, as a readability preference under rule 1; it is not a parsing contract.
5. **Schema evolution** — unknown fields MUST be preserved on round-trip (a proxy or relay receiving an atom with an unknown field MUST re-emit it unchanged). Backward-compatible additions per [11 — Versioning](./11-versioning.md) rely on this.
6. **Atomic-write semantics** — methods that mutate the snapshot MUST be atomic at the wire level — the full request is accepted or rejected; no partial application. In `aala-wire/yaml+git`, one committed operation is one git commit.

## Profile declaration

Every conformant implementation declares its supported wire profiles in the `wire_profiles` field of `Orchestration.capabilities()`. The `OrchestrationCapabilities` shape is normative in [`interfaces/orchestration.md` § `capabilities`](../interfaces/orchestration.md#capabilities); the obligation to expose the declaration is normative in [02 — Conformance § Capability declaration](./02-conformance.md#capability-declaration).

A caller selecting a profile **MUST** inspect `wire_profiles` and pick one before issuing any container call. The chosen profile then governs every subsequent interaction for that session.

## Cross-profile gateways

An implementation MAY operate as a gateway between `rpc`-class profiles — accepting calls in one profile and translating to another. Gateways MUST preserve semantic equivalence (the underlying logical operations are identical) and MUST NOT silently change an error's `(activity, end_state)` pair or classification, nor idempotency guarantees, during translation. Identifiers pass through gateways unchanged (no re-encoding).

A gateway declares both inbound and outbound profiles in its capabilities. Session establishment and the access-control check ([02 — Conformance § Access control](./02-conformance.md#access-control)) are enforced at the gateway boundary, not at the inner profile.

A persistence-class profile is a store, not a call surface: an implementation serving `rpc`-profile calls over a shared `aala-wire/yaml+git` repository is an ordinary implementation, not a gateway.

## What wire profiles do NOT specify

- **Storage backend internals — for `rpc`-class profiles.** An `rpc` profile says how the wire bytes look, not how the implementation stores state internally: an `aala-wire/json+http` implementation may persist atoms in a relational DB, an object store, a graph DB, or anything else. Declaring a persistence-class profile is the deliberate exception — it binds the at-rest representation precisely so the store itself can be shared or exchanged.
- **Performance characteristics.** Profiles bind correctness, not throughput / latency.
- **Internal component organization.** Per the API-only access principle, internals are private.
- **Specific LLM gateway tool choice.** Per [09 — LLM Gateway](./09-llm-gateway.md), tool selection is implementer-chosen and orthogonal to wire profile.
- **Identifier opacity beyond the spec.** Identifiers are normative per [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md) and travel verbatim across all profiles.

## Conformance summary for cross-impl interop

Wire-interoperability is defined once, in [02 — Conformance § Cross-implementation interoperability](./02-conformance.md#cross-implementation-interoperability): a shared profile, the profile's encoding rules honored, identifier formats verbatim, and semantic conformance to chapters 03–12 — all four required. This chapter binds the wire layer (the first three conditions); a shared profile alone is never sufficient.
