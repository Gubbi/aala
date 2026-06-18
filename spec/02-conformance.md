# 02 — Conformance

This chapter defines what makes an implementation **conformant**. The remaining spec chapters describe the rules; this chapter defines what an implementation must satisfy and how it declares its capabilities.

## Single-deployment scope

Conformance is declared and evaluated for **one logical deployment** — a single knowledge layer with its own snapshot space, trees, standard library, and canonical pointer ([00 — Introduction § What's out of scope](./00-introduction.md#whats-out-of-scope)). Every clause in this spec binds within that boundary.

An operator running many logical deployments — a multi-tenant SaaS runs one per tenant — holds N independently conformant deployments behind its own tenancy layer. **Isolation between deployments is a deployment property, not a conformance property**: the contract never has two deployments in view, so no conformance clause requires, or could test, that one deployment's state, events, or telemetry stay invisible to another. Keeping deployments apart is the operator's composition concern, enforced in the layer that establishes sessions and routes callers to deployments ([04 — Snapshots § Session binding](./04-snapshots.md#session-binding)).

## Required containers

The **container registry** — every container in the architecture, with its status — is enumerated in [Appendix A § Containers](./appendix-a-registries.md#containers).

A conformant implementation MUST realize every container the registry marks REQUIRED. It MAY realize the containers the registry marks OPTIONAL; when an optional container is realized, the corresponding interface contract MUST be honored in full.

## Required facet — Projection

[Projection](../interfaces/projection.md) is a cross-cutting **read facet**, not a container. A conformant implementation:

- **MUST** implement the Projection facet on the [Atoms](../interfaces/atoms.md) container (canonical claim prose + glossary — the surface humans review).
- **MAY** implement the Projection facet on other contentful containers (Ingestion, Blast Radius, Hierarchical Navigation). When implemented, the facet contract in [`interfaces/projection.md`](../interfaces/projection.md) MUST be honored in full.
- **MUST** materialize each implemented projection under **exactly one** configured output profile ([Appendix A § Projection output profiles](./appendix-a-registries.md#projection-output-profiles)). The default is `okf` ([Appendix B — OKF Profile](./appendix-b-okf-profile.md)); a deployment **MAY** substitute one custom profile satisfying the profile contract. The configured profile **MUST** be advertised as `projection_profile` in [`Orchestration.capabilities`](../interfaces/orchestration.md#capabilities). Changing the configured profile requires a full re-projection ([Projection § Notes for implementers](../interfaces/projection.md#notes-for-implementers)); no read method selects a profile per call.

Generation and Q&A are **not** aala concerns and impose no conformance obligation: external agents compose answers and documents over aala's read surfaces (see [`docs/analysis/agent-integration-pattern.md`](../docs/analysis/agent-integration-pattern.md)). When Quality is realized, externally composed prose can be submitted for **faithfulness verification** against the atom store ([`quality.md § verify_faithfulness`](../interfaces/quality.md#verify_faithfulness)) — a judgment service, not a generation surface.

## Wire profile declaration

A conformant implementation MUST support at least one wire profile of the **`rpc` class** from the registry in [13 — Wire Profiles](./13-wire-profiles.md#profile-classes) — the class that carries the call surface and the error envelope. It MAY additionally support `persistence`-class profiles (which bind at-rest state and carry no calls), never alone. It MUST declare every profile it supports.

## Capability declaration

Every conformant implementation **MUST** expose `Orchestration.capabilities()`, declaring which containers it realizes (with their interface versions), which containers expose the Projection facet, which output profile those projections are materialized under, and which wire profiles it supports. The `OrchestrationCapabilities` shape, its field semantics, and the content rules — one entry per container in the container registry ([Appendix A § Containers](./appendix-a-registries.md#containers)), `wired: true` exactly when the container is realized, `projection_profile` naming the single configured output profile ([Appendix A § Projection output profiles](./appendix-a-registries.md#projection-output-profiles)) — are normative in [`interfaces/orchestration.md` § `capabilities`](../interfaces/orchestration.md#capabilities).

The declaration **MUST** be stable for the lifetime of the implementation process unless the deployer reconfigures it. A caller **MAY** consult it to decide which features to invoke and which wire profile to use.

## Interface conformance

For each realized container — and for each container exposing the Projection facet — the implementation:

- **MUST** expose every method defined in the corresponding interface file under [`interfaces/`](../interfaces/).
- **MUST** honor every signature exactly: parameter types, return types, and the declared `(activity, end_state)` pairs.
- **MUST** honor the declared idempotency rules.
- **MUST** honor the declared concurrency model (serialized writes, concurrent-safe reads).
- **MUST** enforce each method's declared **Access** operation type via the grant check in [§ Access control](#access-control).
- **MUST** emit every event variant defined in the container's (or facet's) `changes_since` section.
- **MUST** honor the encoding rules of every declared wire profile.
- **MUST** honor the normative identifier formats in [`interfaces/00-shared-types.md`](../interfaces/00-shared-types.md).
- **MAY** expose additional methods, additional fields, or additional event variants. Such extensions MUST NOT change the meaning of existing methods or events.
- **MAY** expose additional registered LLM Gateway use-case keys beyond the standard set ([Appendix A § Use-case keys](./appendix-a-registries.md#use-case-keys); semantics in [09 — LLM Gateway](./09-llm-gateway.md)). All standard keys MUST be present.

## Standard-library registry

A conformant implementation MUST pre-register, per tree, every entry of the standard library enumerated in [Appendix A § Standard library](./appendix-a-registries.md#standard-library): the ten PredicateKindAtoms, the standard-library PredicateAtoms, and the standard-library ClassificationAtoms.

See [03 — Data Model § Standard library](./03-data-model.md#standard-library--pre-registered-atoms-per-tree) for the semantics. Deployments MAY add classifications; they MUST NOT remove or repurpose the standard library.

## Cross-cutting invariants

A conformant implementation MUST honor the cross-cutting rules in the remaining spec chapters:

- **Data Model** ([03](./03-data-model.md)): atom shape, types, schemas, statuses.
- **Snapshots** ([04](./04-snapshots.md)): lifecycle, coherence, isolation.
- **Delta Streams** ([05](./05-delta-streams.md)): ordering, replay equivalence.
- **Conflict Classification** ([06](./06-conflict-classification.md)): outcome semantics, resolution modes.
- **Atom Lifecycle** ([07](./07-atom-lifecycle.md)): transitions and the three cascade channels.
- **Blast Radius** ([08](./08-blast-radius.md)) when wired: pipeline stages, iteration policy.
- **LLM Gateway** ([09](./09-llm-gateway.md)): standard use-case keys.
- **Error Model** ([10](./10-error-model.md)): the three-tier chain, classification, mandated OTel emission, idempotency guarantees.
- **Versioning** ([11](./11-versioning.md)): backward-compatible additions, breaking-change rules.
- **Edge Cases** ([12](./12-edge-cases.md)): concurrent operations, malformed inputs, convergence.
- **Wire Profiles** ([13](./13-wire-profiles.md)): serialization, transport, encoding for the declared profile(s).

## Entailment conformance

A conformant Atoms implementation MUST compute all three entailment tiers (T1, T2, T3) as defined in [03 — Data Model](./03-data-model.md). Implementations MAY choose caching strategy (lazy, eager, or hybrid). The choice MUST NOT affect the set of derived atoms ultimately reachable.

## Cascade consistency (one engine)

The cascade rules ([07 — Atom Lifecycle](./07-atom-lifecycle.md)) are the single normative definition of impact propagation. A conformant implementation:

- **MUST** expose `Atoms.simulate_transition` as the read-only dry-run of the same engine the Status Manager runs for real transitions.
- **MUST** ensure that, for any transition `(atom_id, hypothetical)`, the structural impact set of `simulate_transition` equals the set of atoms a committed `update_status` would transition (same predicted state per atom).
- When Blast Radius is realized, it **MUST** obtain its structural impact from `simulate_transition` and **MUST NOT** re-implement cascade traversal. Its LLM-implicit pass is additive and outside this equivalence.

A reference conformance suite SHOULD verify this by running each planted cascade case through both paths — `simulate_transition` and a committed `update_status` — and asserting the structural sets are identical.

## Error-model conformance (the cross-vendor contract)

The error model ([10 — Error Model](./10-error-model.md)) is the contract that lets independently built containers compose. A conformant container:

- **MUST** produce, for every fallible call, an **Activity Error** drawn from the closed `Activity` vocabulary, terminating in an `end_state` **registered for that activity** ([10 § Activity end-states](./10-error-model.md#activity-end-states)), wrapping a **Primitive Error** as its `cause`. Tier 1 (Operation Error) is produced only at customer touch points and is not required of container internals.
- **MUST** carry the classification **registered for the `(activity, end_state)` pair**: `fix_domain` and `recoverability` equal to the registry entry, and a `severity` that **equals** `derive(fix_domain, recoverability)`. A classification disagreeing with the registry is non-conformant.
- **MUST** confine control-flow-relevant information to the standardized fields (`activity`, `end_state`, `fix_domain`, `recoverability`). It **MUST NOT** require callers to string-match the tier-3 `type` to behave correctly; primitive `type`s are vendor-defined and outside the contract.
- **MUST** set `trace_id` on every Activity (and Operation) Error to the operation's `correlation_id`, return it to the caller, and write it to error logs.
- **MUST** emit each error via OpenTelemetry per [10 § Mandated observability](./10-error-model.md#mandated-observability): one event cascading the `cause` chain, standard field names, the Primitive's detail emitted to telemetry/logs and withheld from end-user renders.
- **MUST NOT** surface a conflict-pipeline outcome ([06](./06-conflict-classification.md)) as an Activity Error, nor vice versa.

A reference conformance suite SHOULD verify that every method's declared activities are raised with valid, derivation-consistent classification, that `trace_id` round-trips from input `correlation_id` to returned error to emitted event, and that a Primitive `cause` is always present.

## Access control

aala's access-control contract is a **grant check at the perimeter** — the deployment surface where a wire profile terminates and sessions are established ([13 — Wire Profiles](./13-wire-profiles.md)). The model below binds *what is checked*. Caller identity, token formats, identity providers, and the role/policy machinery that produces a session's grants are deployment concerns outside this contract ([00 — Introduction § What's out of scope](./00-introduction.md#whats-out-of-scope)).

### Grants

A session ([04 — Snapshots § Session binding](./04-snapshots.md#session-binding)) carries a set of **grants**, each a `(snapshot, operation_type)` pair (shape in [`00-shared-types.md` § Sessions and grants](../interfaces/00-shared-types.md#sessions-and-grants)). The grant set is fixed at session establishment and grows in exactly one in-model way — the creator default below.

Operation types are the Tier-1 **`OperationKind`** vocabulary, enumerated in [Appendix A § Operations (Tier 1)](./appendix-a-registries.md#operations-tier-1) ([10 — Error Model § The standardized Tier-1 vocabulary](./10-error-model.md#the-standardized-tier-1-vocabulary)). One closed registry serves both error framing and access control.

### The check

Every interface method declares **exactly one** operation type in its **Access** line, alongside its **Errors** line in [`interfaces/`](../interfaces/). On admission of every call, the implementation **MUST** verify:

> (*subject snapshot*, the method's declared operation type) ∈ the session's grants

The **subject snapshot** is the calling session's bound snapshot. The snapshot-lifecycle methods are the one exception family: they address snapshots by id ([04 § Cross-snapshot reads](./04-snapshots.md#cross-snapshot-reads)), and their **Access** declarations name the addressed snapshot as the subject — `fork` and `switch_snapshot` check `query` against the source / target (both are read-entry acts: a fork makes the source's content readable in the created snapshot; a rebind makes the target the session's read context), `discard` checks `manage_snapshot` against the named snapshot, and `publish_as_canonical` checks `manage_snapshot` against the snapshot the **canonical pointer currently designates** — the baseline the publish moves the deployment off ([04 § Canonicalization](./04-snapshots.md#canonicalization)). The per-method declarations in [`interfaces/orchestration.md`](../interfaces/orchestration.md) are the binding statements.

A method's Access type usually coincides with the Tier-1 `operation` a touch point frames for it, but the two declarations are independent: `fork` and `switch_snapshot` are framed as `manage_snapshot` operations yet check `query` on their subject snapshot. The check is uniform even for methods reading deployment-level state (usage telemetry, archived evaluation runs): their subject is the session's bound snapshot.

Failures are the two `authorizing` pairs ([10 § Activity end-states](./10-error-model.md#activity-end-states)):

- a call for which no session is established — the establishment context is absent or invalid — **MUST** fail with `authorizing` / `unauthenticated`;
- a call whose session lacks the required grant **MUST** fail with `authorizing` / `denied`.

Both are **blanket pairs**: they apply to every method and are not repeated in per-method **Errors** declarations ([10 § Activity end-states](./10-error-model.md#activity-end-states), notes).

### Enforcement point

The check runs **once, at call admission, at the perimeter** — whichever container's wire surface admitted the call. Containers behind the perimeter **MAY** assume an admitted call is authorized: container-to-container calls within an admitted operation are not re-checked. A cross-profile gateway enforces the check at its own boundary ([13 § Cross-profile gateways](./13-wire-profiles.md#cross-profile-gateways)).

Two methods are **session-floor** surfaces, callable by any established session without a grant: `Orchestration.capabilities` (the deployment composition a caller needs before it can do anything else) and `Orchestration.current_snapshot` (the session's own binding). Every other method requires a grant.

### Defaults

- **Creator grants.** The session that forks a snapshot receives **every operation type** on the created snapshot, atomically with the fork. This is the only in-model grant mint. Whether the deployment extends those grants to the creator's later sessions is grant-producing machinery, out of scope.
- **Canonical grant policy is deployment-defined.** Which sessions hold which operation types on the snapshot the canonical pointer designates — in particular `manage_snapshot`, which gates publish — is the deployment's central access decision.
- **`maintain` and system operations.** Internally-initiated operations (garbage collection, index rebuilds, integrity sweeps) execute in sessions the deployment itself establishes, correlated as `system:`; their grants are deployment-defined. Externally-invoked maintenance methods (e.g. `HierarchicalNav.rebuild`, `Quality.run_benchmark`) require a `maintain` grant on the session's bound snapshot.

### Transport

How establishment context — and with it the grant set — is conveyed is defined **per wire profile** in [13 — Wire Profiles](./13-wire-profiles.md): a Bearer token in the `Authorization` header for `aala-wire/json+http`, connection initialization for `aala-wire/json+mcp`, the medium's own permissions for `aala-wire/yaml+git`. The abstract grant model above is the contract; the wire rows bind only the conveyance.

## API-only access

Every interaction with a container's state MUST go through the container's published interface. An implementation MUST NOT permit external entities (including other containers) to read or mutate a container's internal state directly. This applies at every level — between containers, and between components inside a container.

## Cross-implementation interoperability

Two implementations A and B are **wire-interoperable** if and only if:

1. They share at least one wire profile in `Orchestration.capabilities().wire_profiles`.
2. They both honor the shared profile's encoding rules per [13 — Wire Profiles](./13-wire-profiles.md).
3. They both honor the normative identifier formats verbatim.
4. They both honor every MUST clause in chapters 03–12 (semantic conformance).

Conditions 1–3 are wire-level; condition 4 is semantic-level. All four are required for true interop. Call-level interop requires the shared profile in condition 1 to be `rpc`-class; implementations sharing only a `persistence`-class profile interoperate at the store level — each can read and continue the other's store — but not at the call level ([13 § Profile classes](./13-wire-profiles.md#profile-classes)). A conformant-but-non-interoperable pair (different declared profiles) is permitted by the spec but is a deployment choice, not a conformance violation.

## Conformance test posture

A reference conformance test suite MAY be provided alongside the spec; running it is RECOMMENDED but not required. The binding test is "does the implementation satisfy every MUST in this spec." The test suite, when present, exercises a representative subset.
