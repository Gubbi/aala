# 01 — Conventions

## Normative keywords

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and **OPTIONAL** in this specification are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) \[[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)\] \[[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)\] when, and only when, they appear in all capitals, as shown here.

This declaration covers the entire specification: every file under `spec/` and [`interfaces/`](../interfaces/).

Significance is keyed on capitalization alone. An all-capitals keyword binds whether or not it is bolded — bolding is typography, never significance. Lowercase use of the same words (e.g., "must be careful", "may be returned") carries no normative weight.

## Terminology

| Term | Definition |
|---|---|
| **Atom** | The unit of content in a snapshot. One of five concrete types. See [03 — Data Model](./03-data-model.md). |
| **EntityAtom** | An ordinary subject — a thing the snapshot makes claims about. |
| **RelationAtom** | A single triple connecting two atoms. |
| **ClassificationAtom** | A class of subjects plus the structural schema defining what it means to be classified under it. |
| **PredicateAtom** | A named predicate. Specialized classification whose instances are RelationAtoms. |
| **PredicateKindAtom** | A kind of predicate carrying intrinsic cascade semantics and hard property characteristics. |
| **Predicate kind** | One of the closed set of ten values, organized as inverse pairs and enumerated in [Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds). Each carries hard characteristics + a cascade rule + an `inverse_of` pointer. |
| **Classification schema** | The structural contract on a ClassificationAtom — subject_kind, domain_classifications, attributes, required_relations, optional_relations, max_children, aliases, etc. |
| **Tree** | An autonomous existence — a bounded discourse with its own stakeholders, owners, and lifecycle — identified by `Scope.tree`. Same-claim atoms collapse within a tree; cross-tree similarity surfaces as a candidate `kind=equivalence` link. See [03 — Data Model § Scope](./03-data-model.md#scope). |
| **Derived atom** | An atom whose `derivation` field is populated — produced by entailment from other atoms (T1+T2). |
| **Present state** | An atom status the atom can be in (`active`, `hanging`, `deferred`, `deprecated`). |
| **Removal outcome** | An atom transition that takes the atom out of the snapshot (`superseded`, `duplicate`, `rejected`, `removed`). |
| **Revision** | An in-place content change to an atom — same `AtomId`, recorded as an `Updated` event. Governed by the provenance-fidelity rule in [03 — Data Model § Mutability and revision](./03-data-model.md#mutability-and-revision). |
| **Succession** | Replacement of one claim by a *different* claim: the successor enters as a new atom and the prior atom transitions to `superseded` (gated — compatible refinement only). See [07 — Atom Lifecycle § Revision, succession, and the prior atom's status](./07-atom-lifecycle.md#revision-succession-and-the-prior-atoms-status). |
| **Snapshot** | An addressable, coherent version of all snapshot-bound state. See [04 — Snapshots](./04-snapshots.md). |
| **Session** | The perimeter context every call executes within. A session binds exactly one snapshot at a time — the binding is the sole snapshot selector for reads and writes — and carries the access grants the perimeter checks. See [04 — Snapshots § Session binding](./04-snapshots.md#session-binding) and [02 — Conformance § Access control](./02-conformance.md#access-control). |
| **Grant** | A `(snapshot, operation type)` pair a session carries — the unit of access control, checked at call admission. Operation types are the Tier-1 `OperationKind` vocabulary. See [02 — Conformance § Access control](./02-conformance.md#access-control). |
| **Canonical pointer** | A deployment's single designation of its current canonical snapshot — moved only by the atomic compare-and-swap of publish. See [04 — Snapshots § Canonicalization](./04-snapshots.md#canonicalization). |
| **Container** | A capability unit of the architecture — independently implementable behind its interface contract. The container set and optionality are defined in [02 — Conformance](./02-conformance.md). |
| **Component** | An internal subdivision of a container. Implementation-internal; never part of the binding contract. |
| **Use-case key** | A registered identifier (e.g., `aala.extraction`) that the LLM Gateway routes to a model. See [09 — LLM Gateway](./09-llm-gateway.md). |
| **Delta Stream** | The ordered, append-only event stream each stateful container exposes via `changes_since` / `as_stream` — per-container and per-snapshot. Its typed events are `ChangeEvent`s. This is the formal term throughout the spec for that aspect. Normative semantics in [05 — Delta Streams](./05-delta-streams.md). |
| **Ref** | An opaque checkpoint identifier used to position a consumer in a Delta Stream. Equality is the only client-side operation on refs — stream order is held by the producer ([05 — Delta Streams § Ordering](./05-delta-streams.md#ordering)). |
| **Conformant implementation** | An implementation that satisfies every MUST in this spec. See [02 — Conformance](./02-conformance.md). |
| **Deployment** | One logical aala instance — a single knowledge layer with its own snapshot space, trees, standard library, and canonical pointer. The entire contract is scoped to one deployment; an operator realizes multi-tenancy by running one deployment per tenant ([02 — Conformance § Single-deployment scope](./02-conformance.md#single-deployment-scope)). |
| **Implementer** | The party building an aala implementation. |
| **Deployer** | The party running an aala deployment. |
| **Caller** | Any code (agent, client, container) that invokes an aala API. |

## Notation

Method signatures and type definitions appear in TypeScript-like syntax for readability:

```typescript
function name(arg: Type, optional?: Type): Promise<Result>
```

This describes the contract shape, not a binding implementation language. An implementation MAY use any language whose type system can express the same constraints.

`Promise<T>` denotes an asynchronous return. `Stream<T>` denotes a streaming response.

Optional parameters and fields are marked with `?`.

Union types use `|`. Discriminated unions (variants) use a `kind` field.

Atom status and outcome values appear lowercase in code and field values (e.g., `"active"`, `"superseded"`). Capitalized references in narrative prose (e.g., "the Hanging state") refer to the same underlying value.

## Idempotency and last-write-wins

The contract uses these two terms for distinct guarantees, never one for the other:

**Idempotent.** A method is idempotent when it recognizes a repeated call as a **replay** — the same logical request presented again — and a replay is a **full no-op**: no state mutates, no timestamp moves, no Delta-Stream event is emitted, and the call succeeds, returning the recorded result (or a semantically equivalent one where the original response is no longer retained). How a method recognizes a replay is declared per method in the idempotency registry ([`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency)), in one of three forms:

- **Key-identified** — a declared idempotency key identifies the logical request. A call presenting an already-recorded key is a replay *regardless of differences in its non-key arguments*: the first write stands and its recorded result is returned.
- **State-equality** — the call's entire would-be effect already holds (content equality, a pointer already at its target, an atom already in the named state). No key exists; a repeated call that *changes* values is a new request.
- **Pure recomputation** — the method's result is a pure function of the session's bound snapshot, so repeated calls converge to the same state. Read-only methods are the trivial case.

**Last-write-wins.** An **overwrite** rule, not an idempotency rule: when a later call presents a *different* value for the same target, the later value displaces the earlier one. The later call is a genuine write — it mutates state and emits its events. Last-write-wins applies only where a method's registry row or **Concurrency** declaration states it; it is never implied by "idempotent".

The two compose without tension: a method MAY no-op equivalent repeats (idempotency) *and* overwrite on differing values (last-write-wins) — `BlastRadius.record_resolution` does both. A contract MUST NOT present one guarantee under the other's name.

## Normative ownership

Every concept in this specification has **exactly one normative home**. Every other mention of it, anywhere in the doc set, is an informative cross-reference to that home.

| Concern | Normative home |
|---|---|
| Type shapes, method signatures, per-method **Errors** / **Access** / **Concurrency** declarations, event unions | The container's interface file in [`interfaces/`](../interfaces/) |
| Per-method idempotency contracts | The idempotency registry in [`interfaces/00-shared-types.md § Idempotency`](../interfaces/00-shared-types.md#idempotency); the defining vocabulary is [§ Idempotency and last-write-wins](#idempotency-and-last-write-wins). A method's own **Idempotency** line is an informative restatement of its registry row. |
| Semantics, invariants, lifecycles, ordering, classification rules, conformance | The owning spec chapter in `spec/` |
| Registries (closed enumerations — containers, identifier minting, predicate kinds, the standard library, conflict outcomes, use-case keys, activities, operations, end-states) | [Appendix A — Registries](./appendix-a-registries.md) carries the enumeration; the owning chapter named at the top of each appendix section defines its semantics |

Two prohibitions follow:

- A spec chapter MUST NOT re-declare shapes or signatures; it links the interface file.
- An interface file MUST NOT define cross-cutting semantics; it links the owning chapter.

**Precedence.** Where any duplicated or derived text disagrees with the normative home, the home governs, and the disagreement is a defect in the other text.

[`analysis/`](../analysis/) is informative — forward-facing analyses and comparisons. Nothing outside `spec/` and `interfaces/` carries normative weight.
