# Shared Types

Types and conventions referenced from multiple container interfaces. Read this before the per-container files. The conceptual model is specified in [`spec/03-data-model.md`](../spec/03-data-model.md); this file is the typed shape.

Version: `aala-v0.1`.

Normative keywords in this file — and in every file in this folder — follow BCP 14 (RFC 2119 + RFC 8174) per [01 — Conventions § Normative keywords](../spec/01-conventions.md#normative-keywords): all-capitals occurrences bind; lowercase ones do not.

## Identifiers

All identifiers are strings with a normative format. The format MUST be honored by every conformant implementation. Identifiers are stable: a given id refers to the same logical entity across reads, even across snapshots. Who mints each identifier, the boundary within which it is unique, and how long the name holds are fixed per identifier type by the minting registry in [Appendix A § Identifier minting](../spec/appendix-a-registries.md#identifier-minting); this section owns the formats.

### Format rules

| Field | Format |
|---|---|
| `AtomId` | `atom:<type>:<opaque>` where `<type>` ∈ `{entity, relation, classification, predicate, predicate_kind}` |
| `FragmentId` | `frag:<opaque>` |
| `SnapshotId` | `snap:<opaque>` |
| `Ref` | `ref:<opaque>` |
| `IngestId` | `ingest:<opaque>` |
| `RequestId` | `request:<opaque>` |
| `SystemOpId` | `system:<opaque>` |
| `BlastId` | `blast:<opaque>` |
| `TreeId` | `tree:<opaque>` |
| `CallId` | `call:<opaque>` |

The `<opaque>` portion MUST match the regular expression `[A-Za-z0-9_\-.]+`. Total identifier length MUST be 1–255 characters. Identifiers are case-sensitive. Implementations choose what to put in `<opaque>` — UUID, content hash, sequence, deployment-meaningful slug, etc. The opaque portion is opaque to the contract.

`AtomId`'s `<type>` prefix MUST equal the atom's `type` field. Validators reject mismatches at proposal time; type-checks on referenced atoms (e.g., that an `is_a` target is a ClassificationAtom) reduce to a string-prefix comparison.

### Path-shaped strings (different shape)

These are NOT URN-style identifiers; they have path semantics:

```typescript
type SectionPath   = string  // projection section path (e.g., "domains/payments/services/checkout.md#flow")
type ScopePath     = string  // scope path (e.g., "domains/payments/services")
type ContentKind   = string  // e.g., "slack.message", "meeting.transcript", "doc.adr"
type UseCaseKey    = string  // registered LLM Gateway use case key (e.g., "aala.extraction")
type ProfileKey    = string  // projection output profile (e.g., "okf"); Appendix A § Projection output profiles
```

`SectionPath` and `ScopePath` use `/` as separator and are filesystem-friendly. `ContentKind` and `UseCaseKey` use `.` as the namespace separator and are already prefixed by convention. `ProfileKey` names the single output profile a deployment's projections are materialized under ([Appendix A § Projection output profiles](../spec/appendix-a-registries.md#projection-output-profiles)); the built-in default is `okf`.

### TypeScript declarations

```typescript
type AtomId        = string  // "atom:<type>:<opaque>"
type FragmentId    = string  // "frag:<opaque>"
type SnapshotId    = string  // "snap:<opaque>"
type Ref           = string  // "ref:<opaque>"
type IngestId      = string  // "ingest:<opaque>"
type RequestId     = string  // "request:<opaque>"
type SystemOpId    = string  // "system:<opaque>" — internally-initiated operation (rebuild / sweep / maintenance)
type BlastId       = string  // "blast:<opaque>"
type TreeId        = string  // "tree:<opaque>"
type CallId        = string  // "call:<opaque>" — one LLM Gateway call's structured observability record (Appendix A § Identifier minting)
type CorrelationId = string  // an IngestId, RequestId, or SystemOpId — identifies the originating operation; opaque beyond the prefix
type SectionPath   = string  // path-shaped
type ScopePath     = string  // path-shaped
type ContentKind   = string  // namespaced (e.g., "slack.message")
type UseCaseKey    = string  // namespaced (e.g., "aala.extraction")
type ProfileKey    = string  // projection output profile (e.g., "okf")
```

## Timestamps and durations

Every timestamp-valued `string` field in this contract (`timestamp`, `extracted_at`, `emitted_at`, `changed_at`, `created_at`, `opened_at`, `resolved_at`, `effective_from` / `effective_to`, `since`, …) MUST carry an [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) `date-time` value with an explicit UTC offset (`Z` or `±hh:mm`); offset-less local times are not valid. Inline `// RFC 3339` comments on fields reference this rule.

Schema attribute values follow the same profile: a value typed `datetime` ([§ Schemas](#schemas-carried-on-classification-and-predicate-atoms)) is an RFC 3339 `date-time`, a value typed `date` is an RFC 3339 `full-date`, and a value typed `duration` is an ISO 8601 duration string (e.g., `"PT30S"`) — durations have no RFC 3339 profile.

## NormalizedFragment

The shape Ingestion produces and Atoms consumes.

```typescript
interface NormalizedFragment {
  message_id:    string                  // adapter-minted idempotency key, unique across the deployment's sources (Appendix A § Identifier minting)
  sender:        string
  recipients:    string[]
  payload:       string                  // the prose / message body
  timestamp:     string                  // RFC 3339 (§ Timestamps and durations)
  content_kind:  ContentKind
  source_uri:    string                  // back-reference to source artifact
  extras:        Record<string, unknown> // source-specific structure
}
```

## Atom (base shape)

Every atom inherits this shape. Concrete subtypes are `EntityAtom`, `RelationAtom`, `ClassificationAtom`, `PredicateAtom`, and `PredicateKindAtom`.

```typescript
type AtomType =
  | "entity"
  | "relation"
  | "classification"
  | "predicate"
  | "predicate_kind"

type AtomStatus =
  | "active"
  | "hanging"
  | "deferred"
  | "deprecated"
  | "superseded"        // removal outcomes — a committed atom retains these as a tombstone (spec/07)
  | "duplicate"
  | "rejected"
  | "removed"

interface StatusMeta {
  superseded_by?:       AtomId   // REQUIRED when state = "superseded" — the successor; asserted references resolve through it (spec/07 § Observable resolution invariant)
  duplicate_of?:        AtomId   // REQUIRED when state = "duplicate"
  blast_id?:            BlastId  // present iff a Blast Radius resolution drove the atom to this state — stamped by the resolution mapping (interfaces/blast-radius.md § record_resolution)
  outcome_kind?:        ConflictOutcomeKind     // present iff a conflict-pipeline resolution drove the atom to this state (spec/06 § Outcome recording); enum in interfaces/atoms.md, registry in spec Appendix A § Conflict outcomes
  outcome_stage?:       number | "entailment"   // the pipeline stage that detected the situation (spec/06 § Pipeline stages); present iff outcome_kind is
  outcome_confidence?:  number                  // 0.0–1.0 match confidence, for similarity-driven classifications (e.g., Duplicate)
}
// blast_id / outcome_kind / outcome_stage / outcome_confidence are non-logical audit metadata — they
// record what drove the transition, never drive reasoning. Conflict situations themselves live in
// atom-space (spec/06).

interface Status {
  state:           AtomStatus
  correlation_id:  CorrelationId                 // operation that drove the atom to this state; equals the correlation_id of the change event that set it. Non-logical audit metadata.
  reason?:         string                        // human-readable rationale for the current state
  changed_at?:     string                        // RFC 3339; REQUIRED when state is not the initial "active"
  meta?:           StatusMeta                    // removal-outcome fields + transition-provenance audit metadata (see StatusMeta)
}

interface Atom {
  id:            AtomId                          // "atom:<type>:<opaque>" — type prefix MUST match `type` field
  type:          AtomType
  subject:       string                          // prose name OR AtomId (per concrete type / schema's subject_kind)
  is_a?:         AtomId                          // primary classification (entity/relation) or superclass (classification/predicate); see per-type rules
  attributes:    Record<string, unknown>         // VALUES for the typed slots its classification declares (schema.attributes); validated structurally, excluded from logical reasoning
  annotations:   Record<string, unknown>         // free-form record metadata; never schema-validated, excluded from logical reasoning (AnnotationProperty analog)
  sources:       Source[]
  confidence:    number                          // 0.0–1.0; non-logical
  scope:         Scope
  status:        Status
  derivation?:   Derivation                      // present iff this atom is derived (not asserted)
}

interface Derivation {
  from:   AtomId[]                              // premises this atom was derived from
  rule:   string                                // entailment rule name (e.g., "is_a-transitive", "predicate-chain")
}
```

The `id` field's type prefix MUST equal the `type` field. Validators MUST reject atoms where the two disagree.

## Concrete atom types

```typescript
interface EntityAtom extends Atom {
  type:    "entity"
  is_a:    AtomId                               // REQUIRED; points to a ClassificationAtom
  // subject is prose or AtomId per the classification's schema.subject_kind
}

interface RelationAtom extends Atom {
  type:    "relation"
  is_a:    AtomId                               // REQUIRED; points to a PredicateAtom
  subject: AtomId                               // from-atom (AtomId)
  object:  AtomId                               // to-atom (AtomId)
}

interface ClassificationAtom extends Atom {
  type:    "classification"
  is_a?:   AtomId                               // subclass-of, same type-domain; REQUIRED except the single entity-domain root (Classification("Thing")), which omits it
  schema:  ClassificationSchema
}

interface PredicateAtom extends Atom {
  type:    "predicate"
  is_a:    AtomId                               // REQUIRED; points to a PredicateKindAtom OR another PredicateAtom (for sub-predicate hierarchies)
  schema:  PredicateSchema                      // domain/range/characteristics; layered on top of ClassificationSchema features
}

interface PredicateKindAtom extends Atom {
  type:    "predicate_kind"
  // no is_a — the ten kinds are flat roots; membership is carried by `type`
  kind:    PredicateKind                        // closed enum
  semantics: PredicateKindSemantics             // hard characteristics + cascade rules
}
```

## Schemas (carried on classification and predicate atoms)

```typescript
type SubjectKind = "named" | "referential"

type ValueType =
  | "string"
  | "number"
  | "integer"
  | "boolean"
  | "date"
  | "datetime"
  | "duration"
  | "uri"
  | "structured"

interface AttributeSchema {
  key:        string
  value_type: ValueType
  required:   boolean
}

interface RelationConstraint {
  predicate:               AtomId               // PredicateAtom
  target_classification:   AtomId               // ClassificationAtom
  min:                     number
  max:                     number | "unbounded"
}

interface ClassificationSchema {
  subject_kind?:           SubjectKind          // absent = abstract (root/intermediate spanning both); once set, fixed for all descendants
  domain_classifications?: AtomId[]             // referential classifications only: allowed classes of the subject reference (the referent). Absent/empty = open. N/A for named.
  attributes:              AttributeSchema[]
  required_relations:      RelationConstraint[]
  optional_relations:      RelationConstraint[]
  max_children:            number               // default 20
  aliases:                 string[]
  see_also:                AtomId[]
  canonical_for?:          string
  disjoint_with:           AtomId[]             // within-tree
  definition?:             string               // prose definition for glossary
}

type PredicateKind =
  // self-inverse
  | "association"
  | "equivalence"
  // specialization ⟷ generalization
  | "specialization"
  | "generalization"
  // aggregation ⟷ member_of
  | "aggregation"
  | "member_of"
  // composition ⟷ component_of
  | "composition"
  | "component_of"
  // dependency ⟷ enablement
  | "dependency"
  | "enablement"

interface PropertyCharacteristics {
  symmetric:            boolean | "inherited"
  asymmetric:           boolean | "inherited"
  irreflexive:          boolean | "inherited"
  transitive:           boolean | "inherited"   // never true for the composition family — composition/component_of are not transitive (spec/03 § Predicate kinds); transitive containment derives only through declared chain predicates
  functional:           boolean                 // extractor-settable; "true" hard on component_of (single-whole rule)
  inverse_functional:   boolean                 // extractor-settable; "true" hard on composition (single-parent rule)
}
// Cardinality-like characteristics (functional, inverse_functional) are validated over ASSERTED edges only;
// derived edges never count toward them (spec/03 § Predicate kinds).

interface ChainStep {
  predicate:  AtomId                            // a concrete PredicateAtom — never a PredicateKindAtom; matched by the named predicate or any sub-predicate of it
  repeat?:    "+"                               // one or more consecutive edges of this step's predicate; absent = exactly one
}

interface PredicateSchema {
  // PredicateAtom is a specialized ClassificationAtom; it inherits the classification-shaped fields:
  subject_kind:            SubjectKind          // always "named" for predicates
  attributes:              AttributeSchema[]
  max_children:            number
  aliases:                 string[]
  see_also:                AtomId[]
  disjoint_with:           AtomId[]
  definition?:             string

  // Predicate-specific:
  domain_classifications:  AtomId[]             // allowed classes of the subject reference (the RelationAtom's from-atom) — same field generalized on ClassificationSchema
  range_classifications:   AtomId[]             // valid target-atom (object) classifications
  characteristics:         PropertyCharacteristics
  inverse_of?:             AtomId               // paired PredicateAtom
  chain?:                  ChainStep[]          // chain-derivation declaration; not inherited by sub-predicates. MUST NOT be declared on a composition-family predicate. Semantics: spec/03 § Chain derivation
}

interface PredicateKindSemantics {
  hard_characteristics:    PropertyCharacteristics   // per-kind values fixed by spec/03 § Predicate kinds; composition/component_of exclude `transitive`
  cascade:                 CascadeRule
  inverse_of:              PredicateKind        // the inverse kind (self for the symmetric kinds: association, equivalence)
  conflict_outcomes:       string[]             // per-kind conflict outcome names (informational)
}

interface CascadeRule {
  // When source atom (the "from" of a RelationAtom of this kind) transitions to one of these statuses,
  // the target atom (the "to") transitions to "hanging".
  source_triggers_target_hanging:   AtomStatus[]

  // When target atom transitions to one of these statuses, the source atom transitions to "hanging".
  target_triggers_source_hanging:   AtomStatus[]

  // For composition: cascade only fires when the triggering atom's confidence ≥ threshold (default 0.8, deployment-overridable).
  confidence_threshold?:            number
}
// The trigger arrays for the built-in kinds are fixed by spec/03 § Predicate kinds and spec/07 § Cascade rules:
// dependency/enablement list every non-"active" status (uniform with the scope-premise channel);
// composition/component_of list the removal outcomes only, with confidence_threshold set;
// all other kinds (including equivalence, which never transmits a cascade) carry empty arrays.
// A "superseded" trigger transitions nothing in practice: incident references are evaluated against
// the active successor (spec/07 § Observable resolution invariant), so gated succession is silent.
```

## Source

```typescript
interface Source {
  uri:           string                          // source artifact URI
  quote:         string                          // supporting text from the source
  extracted_at:  string                          // RFC 3339
}
```

Every atom extracted from a fragment has at least one source. Sources accumulate through in-place revision ([spec/03 § Mutability and revision](../spec/03-data-model.md#mutability-and-revision)); an atom entering as a refinement or successor of another may inherit the prior atom's sources and append its own.

## Scope

```typescript
interface Scope {
  tree:              TreeId                     // bounded discourse — Duplicate detection runs within-tree only
  environment?:      string
  time_scope?:       TimeScope
  constraint_atoms:  AtomId[]                   // ConstraintAtoms gating this atom (logical "premise" cascade)
}

interface TimeScope {
  effective_from?:   string                     // RFC 3339
  effective_to?:     string                     // RFC 3339
}
```

`tree` is part of the logical substrate. `time_scope` is non-logical metadata.

## Delta-Stream events

Every stateful container exposes its own Delta Stream via `changes_since(ref)` (incremental deltas of the current snapshot — the calling session's bound snapshot, [04 — Snapshots § Session binding](../spec/04-snapshots.md#session-binding)) and `as_stream()` (full current-snapshot state from empty, for cold-start consumers). Both emit the same union-typed events; specific variants are listed in each container's interface file.

The shared envelope:

```typescript
interface ChangeEvent<TVariant> {
  ref:             Ref                          // checkpoint after applying this event — opaque; equality is the only client-side comparison (spec/05 § Ordering)
  kind:            string                       // event kind, e.g., "Added", "StatusChanged"
  payload:         TVariant                     // kind-specific payload
  emitted_at:      string                       // RFC 3339
  source_ref?:     Ref                          // upstream stream checkpoint that triggered this event (positional, stream-pair-local)
  correlation_id:  CorrelationId                // the originating operation (ingest:/request:/system:); always present; propagated across all containers for cross-stream grouping & debugging
}

interface ChangesPage<TVariant> {
  events:        ChangeEvent<TVariant>[]
  next_ref?:     Ref                            // absent ⇒ caller is caught up (for as_stream: the bootstrap is complete)
}

function changes_since(
  ref:    Ref | null,   // caller's cursor; null = from the start of THIS working snapshot's own Delta Stream (its operations). Distinct from each ChangeEvent.ref.
  limit?: number        // optional soft cap on events per page
): Promise<ChangesPage<unknown>>

function as_stream(
  ref?:   Ref | null,   // bootstrap cursor from a previous as_stream page; absent/null = start the bootstrap from empty
  limit?: number        // optional soft cap on events per page
): Promise<ChangesPage<unknown>>   // full current-snapshot state from empty (parent-canonical materialization + this snapshot's deltas) — pages like changes_since
```

The `ref` argument is the caller's cursor (last consumed checkpoint), not a `ChangeEvent.ref`. `ref = null` returns events from the beginning of the **current working snapshot's own Delta Stream** — only the operations that occurred in it, layered on top of its parent (the post-fork delta), not a reconstruction of inherited state. A consumer holding no inherited state instead calls `as_stream()`, which reconstructs the full current state from empty (replays the parent canonical's materialization, then this snapshot's deltas) and is resumable through its own `next_ref` cursors; on completion the consumer continues incrementally with `changes_since` from the bootstrap's final event ([05 § Full-state stream](../spec/05-delta-streams.md#full-state-stream)). The page size cap is implementation-specific; callers keep paging on `next_ref` until it is absent. Refs are opaque — consumers never order-compare them; positioning is server-side ([05 § Ordering](../spec/05-delta-streams.md#ordering)). Full normative semantics in [05 — Delta Streams](../spec/05-delta-streams.md).

**Errors:** `querying`; both methods additionally raise `stale_ref` when a passed `ref` does not name a checkpoint delivered by the calling session's bound snapshot — foreign, discarded, or unrecognized ([05 § Cross-snapshot behavior](../spec/05-delta-streams.md#cross-snapshot-behavior)).

**Access:** `query` ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)).

These shared declarations apply to every container's implementation of the two methods.

## Paged collection reads

Delta Streams page with `ChangesPage` / `next_ref`; every other unbounded collection read pages with the same idiom, through one shared envelope:

```typescript
type Cursor = string                   // opaque continuation token; obtained only from a previous Page

interface PageRequest {
  cursor?:  Cursor                     // absent = start the enumeration from the beginning
  limit?:   number                     // optional soft cap on items per page (the server MAY return fewer)
}

interface Page<T> {
  items:        T[]
  next_cursor?: Cursor                 // absent ⇒ the enumeration is complete
}
```

Every collection read whose result set is unbounded — its cardinality grows with the knowledge base — takes `page?: PageRequest` as its **final parameter** and returns `Promise<Page<T>>`. Calling without `page` returns the first page at an implementation-default size; the caller passes each page's `next_cursor` as the next call's `cursor` and keeps paging until `next_cursor` is absent — exactly the `ChangesPage` contract. Per-page size caps live in `PageRequest.limit` only; filter shapes carry no `limit` field.

**Enumeration order.** Each paged method enumerates its collection in an implementation-chosen total order that MUST be stable while the underlying state is unchanged. Against unchanged state, a full enumeration — first page through cursor exhaustion — MUST yield every item exactly once: no omissions, no duplicates. The order itself is not part of the cross-implementation contract unless a method states one.

**Interleaved mutation.** A multi-page enumeration spans operations and is **weakly consistent**: an item present for the whole enumeration MUST appear exactly once; an item added or removed mid-enumeration MAY appear or be missed; no item appears twice. An implementation that cannot resume a cursor after intervening writes MAY instead fail it as invalid (below); the caller restarts the enumeration.

**Cursor validity.** A `Cursor` is opaque; the caller MUST pass it back unchanged. It is valid only for the method that produced it, called with the same non-page arguments, against the snapshot it was produced against. A cursor that is malformed, foreign (a different method, argument set, or snapshot), or no longer resumable fails with the method's declared malformed-request pair — `querying` / `invalid_query`, `navigating` / `invalid_query`, or `evaluating` / `invalid_input` ([Appendix A § Activity end-states](../spec/appendix-a-registries.md#activity-end-states)); the caller restarts from the first page.

**Bounded-by-design reads are exempt.** A collection read whose signature takes no `page` parameter returns a plain array because its cardinality is capped by something other than knowledge-base size — a governed registry, one node's fan-out, or a top-k result: `Orchestration.list_trees` and `capabilities`, `HierarchicalNav.axes`, `Atoms.list_children` and `resolve_subject`, `LLMGateway.registered_use_cases` and `routing`, `Quality.metrics` (bucket count is bounded by the grouping dimensions) and `list_golden_sets` (a deployer-curated registry). Single-record reads (`get_report`, `golden_set_results`, …) are not collection reads and never page.

## Cascade paths

```typescript
interface CascadeStep {
  channel:         "predicate-kind" | "scope-premise" | "derivation-invalidation"
  edge_chain:      AtomId[]                     // the asserted edges walked, origin → impacted atom
  predicate_kind?: PredicateKind                // present when channel = "predicate-kind"
  rule?:           string                       // present when channel = "derivation-invalidation"
}
```

One step of a cascade path — how the engine reached an impacted atom from the origin. Produced by [`Atoms.simulate_transition`](./atoms.md#simulate_transition) (the cascade engine's dry-run) and embedded unchanged in Blast Radius reports ([`BlastImpact.via`](./blast-radius.md#types)). Channel semantics are normative in [07 — Atom Lifecycle § Cascade rules](../spec/07-atom-lifecycle.md#cascade-rules).

## SnapshotRef

The handle the Orchestration container exposes for snapshots; consumed by every other container.

```typescript
interface SnapshotRef {
  id:            SnapshotId
  parent?:       SnapshotId                     // forked from this snapshot
  state:         "working" | "canonical" | "discarded"
  created_at:    string
}
```

## Errors — the three-tier model

Every fallible call produces an error structured as a chain of up to three tiers. There is **no catch-all error union**; each tier is a defined shape, and callers act on the tier appropriate to their audience. The binding semantics are in [10 — Error Model](../spec/10-error-model.md); the shapes are here.

```typescript
// ── Classification (declared once at the Activity tier, propagated upward) ──
type FixDomain     = "client" | "data" | "infra" | "aala"
//   client  — fix is in the caller's input or usage
//   data    — fix requires repairing stored content (corrupt/malformed atom, snapshot)
//   infra   — fix is in the underlying infrastructure (permissions, disk, network, provider)
//   aala    — fix is in an aala container's code (invariant violated)

type Recoverability = "retryable" | "user_action" | "operator_action" | "not_recoverable"
//   retryable        — transient; retry the same call after backoff
//   user_action      — caller must change input/usage, then retry
//   operator_action  — operator must repair infra or data, then retry
//   not_recoverable  — invariant violated; escalate to a developer

type Severity = "warn" | "error"
// DERIVED, never declared — a function of the pair (fix_domain, recoverability). See 10 — Error Model § Severity.
//   retryable / user_action → always "warn"; operator_action / not_recoverable → "error",
//   EXCEPT fix_domain = "client" which stays "warn" (caller misuse, e.g. not_wired — nothing is broken).

// ── Tier 3 — Primitive Error: the atomic technical leaf. Implementation-specific;
//    NOT part of the cross-vendor contract. Reserved for the technical audience.
//    Always carried as the `cause` of an Activity Error; surfaced to telemetry + error logs, never to the end user. ──
interface PrimitiveError {
  tier:       "primitive"
  type:       string                       // stable identifier = the leaf type name (snake_case); vendor-defined
  message:    string                       // technical explanation of the leaf failure
  location?:  string                       // code or domain location (file:line, atom path, asset key)
  details?:   Record<string, unknown>      // typed leaf fields (namespaced in emission as aala.error.<type>.<field>)
}

// ── Tier 2 — Activity Error: what the system was doing for the user, and that it didn't work.
//    Standardized cross-vendor contract. Every container emits these. Carries classification. ──
type Activity =
  | "ingesting" | "extracting" | "classifying_conflicts" | "transitioning" | "revising"
  | "snapshotting" | "managing" | "querying" | "navigating" | "analyzing_impact"
  | "evaluating" | "generating" | "authorizing"

interface ActivityError {
  tier:            "activity"
  activity:        Activity                // the user-meaningful unit of work — the headline
  end_state:       string                  // which error end-state the activity terminated in — closed per activity; see 10 § Activity end-states
  message:         string                  // "what happened," phrased for the person trying to get something done
  fix_domain:      FixDomain               // fixed per (activity, end_state) by the spec registry
  recoverability:  Recoverability          // fixed per (activity, end_state) by the spec registry
  severity:        Severity                // MUST equal derive(fix_domain, recoverability)
  trace_id:        CorrelationId           // the operation's correlation_id, carried as an error field; emitted to telemetry as the aala.correlation_id attribute — never as the W3C trace-id (10 § Mandated observability)
  component?:      string                  // which container/component — for "where to look"; for logs/telemetry, not the headline
  hint?:           string                  // optional fixed corrective guidance
  retry_after_ms?: number                  // present when recoverability = "retryable" and a delay is known
  cause:           PrimitiveError          // REQUIRED — the evidence; emitted to telemetry/logs, filtered from user-facing render
}

// ── Tier 1 — Operation Error: the whole job the user asked for. Owned by customer touch points
//    (the product/integration surface), not by the system internals. Optional; present only at the edge. ──
type OperationKind =
  | "ingest" | "query" | "review" | "manage_snapshot" | "manage_tree" | "maintain"

interface OperationError {
  tier:            "operation"
  operation:       OperationKind           // the job, in client intent
  message:         string                  // end-user framing
  fix_domain:      FixDomain               // delegated from the Activity
  recoverability:  Recoverability          // delegated from the Activity
  severity:        Severity                // delegated from the Activity
  trace_id:        CorrelationId
  cause:           ActivityError           // the activity that failed
}
```

**Reading the chain.** Callers branch on `recoverability` for control flow (*retry / surface / alert / abort*) — never on tier-3 type strings. Where behavior genuinely differs by case beyond classification, the discriminant is the `(activity, end_state)` pair — both closed vocabularies, with classification fixed per pair by the registry in [10 § Activity end-states](../spec/10-error-model.md#activity-end-states). The `activity` + `message` (+ optional `hint`) are for human communication; `component` is "where to look"; the `cause` Primitive is for the technical audience via telemetry and logs. A method's **Errors** declaration names the `(activity, end_state)` pairs it can raise — that is the closed contract; primitive `type`s are vendor-defined and not enumerated by the spec.

## Sessions and grants

Every call executes within a session ([04 — Snapshots § Session binding](../spec/04-snapshots.md#session-binding)), and access control is a grant check the perimeter runs at call admission ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)). A grant pairs a snapshot with an operation type from the Tier-1 `OperationKind` vocabulary above:

```typescript
interface Grant {
  snapshot:        SnapshotId
  operation_type:  OperationKind     // the Tier-1 vocabulary — one closed registry for error framing and access control
}
```

A session carries `grants: Grant[]` — fixed at establishment and extended only by the creator default: the session that forks a snapshot receives all six operation types on the created snapshot ([02 § Access control](../spec/02-conformance.md#access-control)). There is no session object on the wire: the perimeter holds the session context — the snapshot binding plus the grant set — and how that context is conveyed is bound per wire profile ([13 — Wire Profiles](../spec/13-wire-profiles.md)). Each interface method declares the operation type its check uses in its **Access** line, alongside its **Errors** line.

## Idempotency

This registry is the **single normative home** for per-method idempotency contracts ([01 — Conventions § Normative ownership](../spec/01-conventions.md#normative-ownership)). The vocabulary — what **idempotent** and **last-write-wins** mean, and that they are distinct guarantees — is defined in [01 — Conventions § Idempotency and last-write-wins](../spec/01-conventions.md#idempotency-and-last-write-wins): an idempotent **replay** is a *full no-op* (no state mutation, no timestamp movement, no Delta-Stream event; the recorded or a semantically equivalent result is returned), and last-write-wins is an *overwrite* contract a genuine second write executes under. A method's own **Idempotency** line restates its row here for reading convenience; where the two disagree, this registry governs.

| Method | Replay recognition | Contract |
|---|---|---|
| `Ingestion.accept(fragment)` | key: `fragment.message_id`, accepted fragments only | A reused id of an **accepted** fragment is a replay regardless of non-key differences — full no-op; the recorded `AcceptResult` (or a semantically equivalent one) is returned. A rejection persists nothing and leaves no replay record: re-submitting a rejected `message_id` is a fresh evaluation that **MAY** accept ([`ingestion.md § accept`](./ingestion.md#accept)). |
| `Atoms.propose(fragment, hints?)` | key: `fragment.message_id`, within the bound snapshot | A reused id is a replay regardless of non-key differences — full no-op; the recorded `ProposeResult` (or a semantically equivalent one) is returned. |
| `Orchestration.ingest(fragment, options?)` | key: `fragment.message_id` | **Retry-safe by composition:** a retry replays completed per-container steps as no-ops and resumes incomplete ones ([10 § Cross-container failures](../spec/10-error-model.md#cross-container-failures)). |
| `Atoms.update_status(atom_id, outcome, rationale, meta?)` | key: `(atom_id, outcome, meta.idempotency_key?)` | An equivalent replay — notably a call naming the atom's current state — is a full no-op: state, `status.changed_at`, and the Delta Stream are untouched ([07 § API for transitions](../spec/07-atom-lifecycle.md#api-for-transitions)). A call naming a *different* outcome is an ordinary transition under the lifecycle rules — never an overwrite. |
| `Atoms.update(atom_id, changes, rationale)` | state-equality | A call whose `changes` leave every targeted field at its current value is a full no-op. No key exists; a repeated call that changes values is a new revision. |
| `BlastRadius.analyze(input)` | key: the input tuple `(atom_id, hypothetical, depth_limit, scope_filter)`, while atom state is unchanged | Replay returns the existing open report — same `blast_id`, no second `ReportOpened`. Once atom state has changed since generation, a re-run is a new request and opens a fresh report ([08 § Determinism and report identity](../spec/08-blast-radius.md#determinism-and-report-identity)). |
| `BlastRadius.record_resolution(blast_id, atom_id, resolution, rationale, meta?)` | state-equality per `(blast_id, atom_id)` | Re-recording the already-recorded `resolution` + `rationale` + `meta` is a full no-op, recognized **before** the report-level checks — a replay against a since-closed report succeeds. A *differing* re-resolution is not a replay: it overwrites under **last-write-wins** per `(blast_id, atom_id)` — a genuine write that issues the newly mapped `update_status`, mutates the report, and emits `ImpactResolved`; it stands only if that transition completes (apply-then-record — [`blast-radius.md § record_resolution`](./blast-radius.md#record_resolution)). (Also reachable via `Orchestration.record_resolution`, which forwards unchanged.) |
| `Quality.record_feedback(call_id, feedback)` | state-equality per `call_id` | Re-recording identical feedback is a full no-op. Differing feedback overwrites under **last-write-wins** per `call_id`. |
| `Quality.add_golden_case(set_id, case_)` | key: `case_.case_id` | A reused case id is a replay regardless of non-key differences — full no-op; the recorded case stands. |
| `Quality.enable_shadow_eval(use_cases?, sample_rate?)` | state-equality | Re-applying the current configuration is a full no-op. A differing configuration overwrites under **last-write-wins** — one shadow-eval configuration exists at a time ([`quality.md § enable_shadow_eval`](./quality.md#enable_shadow_eval)). |
| `Orchestration.publish_as_canonical(snapshot_id)` | state-equality at the canonical pointer | Publishing the snapshot the pointer already designates is a no-op success returning its `SnapshotRef` ([04 § Canonicalization](../spec/04-snapshots.md#canonicalization)). |
| `Orchestration.register_tree(tree, display_name, description?)` | state-equality per `tree` | Re-registering an existing tree with identical `display_name` / `description` is a replay — full no-op returning the existing `TreeInfo`, no event. A registration naming an existing `tree` with *differing* fields is not a replay: it fails with `managing` / `already_registered` ([`orchestration.md § register_tree`](./orchestration.md#register_tree)). |
| `Orchestration.update_tree(tree, display_name?, description?)` | state-equality | Re-applying the current display values — or supplying none — is a full no-op; no event is emitted. |
| `HierarchicalNav.rebuild(axis_id?, scope?)` | pure recomputation | The rebuilt state is a pure function of the bound snapshot; repeated calls converge to the same tree state. |
| `LLMGateway.chat(...)` | conditional: `options.cache_key` hit | Not idempotent in general (model outputs are non-deterministic). When `cache_key` is supplied and a cache hit is taken, the cached response is returned unchanged. |

Two blanket rules close the registry:

- **Read-only methods are naturally idempotent** — pure recomputation over the bound snapshot — and carry no rows here.
- **A mutating method with no row carries no idempotency contract**: a repeated call is a new call. Callers MUST reason about partial-failure scenarios before retrying ([10 § Idempotency contracts](../spec/10-error-model.md#idempotency-contracts)).

## Concurrency

Every call executes within a **session** bound to exactly one snapshot at a time; the binding is the sole selector of the snapshot a read or write executes against — no method takes a per-call snapshot parameter ([04 — Snapshots § Session binding](../spec/04-snapshots.md#session-binding)). Unless otherwise stated:

- **Reads** against a snapshot are concurrent-safe and serializable with each other.
- **Writes** against a snapshot are serialized per snapshot at **operation grain** — one operation (an `ingest` / `propose` / `update` / `update_status`, including the conflict classification, cascade to fixpoint, and entailment recompute it triggers) commits as a unit before the next begins. Concurrent writes to the same snapshot — from one session or several bound to it — are either queued (blocking) or fail with an Activity Error carrying `end_state: "contended"` (`infra`/`retryable`; caller retries). The choice is per-implementation; the contract is "no partial / interleaved writes." Concurrency across operations is achieved by concurrent sessions bound to separate snapshots, not by interleaving within one. (`BlastRadius.analyze` is read-only and outside this rule; `BlastRadius.record_resolution` mutates only the report store — the atom transition it drives is an ordinary `update_status` write under this rule.)
- **Snapshot lifecycle operations** (forking, publishing, discarding) are atomic at the snapshot level — observers see either the pre or the post state of a snapshot lifecycle event, never an intermediate. Publish is an atomic compare-and-swap on the canonical pointer ([04 § Canonicalization](../spec/04-snapshots.md#canonicalization)). `switch_snapshot` rebinds the calling session atomically: a call executes wholly against the binding in effect at its admission.

## Versioning

Each container interface declares a `version` string at the top of its file (e.g., `aala-v0.1`). The version covers method signatures + declared `(activity, end_state)` pairs + declared **Access** operation types + idempotency registry rows ([§ Idempotency](#idempotency)) + event kinds. A **backward-compatible** change adds optional fields or new variants without removing or repurposing existing ones; a **breaking** change does not. Pre-publish, in-place overwrite is permitted at the `v0.x` level.
