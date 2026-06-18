# Interface Contracts

Formal interface contracts for each container, plus the cross-cutting projection read facet. These are the binding agreements at aala's boundaries — anything you can call across a container boundary in aala lives here, with exact signatures, error model, idempotency rules, and concurrency semantics.

## Audience

- **Implementer** writing a concrete realization of a container (or all of aala).
- **Spec author** drawing on these to assemble `docs/spec/` — the consolidated formal specification.
- **Reader** comparing implementations or building a client.

## What's here

| File | Covers |
|---|---|
| [`00-shared-types.md`](./00-shared-types.md) | Identifiers, common data types, the three-tier error shapes, event types — shared across multiple containers. Read this first. |
| [`ingestion.md`](./ingestion.md) | Ingestion (non-optional) |
| [`atoms.md`](./atoms.md) | Atoms (non-optional). The central interface; many others reference it. |
| [`projection.md`](./projection.md) | Projection — the cross-cutting read facet each contentful container implements (required of Atoms; optional elsewhere) |
| [`orchestration.md`](./orchestration.md) | Orchestration (non-optional) — the outside-world entry point |
| [`hierarchical-nav.md`](./hierarchical-nav.md) | Hierarchical Navigation (optional) |
| [`blast-radius.md`](./blast-radius.md) | Blast Radius (optional) |
| [`llm-gateway.md`](./llm-gateway.md) | LLM Gateway (cross-cutting infrastructure) |
| [`quality.md`](./quality.md) | Quality (optional cross-cutting measurement) |

## Conventions

- **Normative keywords** — the key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, NOT RECOMMENDED, MAY, and OPTIONAL in every file under `docs/interfaces/` are to be interpreted as described in BCP 14 (\[[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)\] \[[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)\]) when, and only when, they appear in all capitals. The declaration in [01 — Conventions § Normative keywords](../spec/01-conventions.md#normative-keywords) covers this folder; lowercase use of the same words carries no normative weight.
- **Signature syntax** is TypeScript-like for readability. The shape is what's binding, not the syntax — implementations may use any language whose types can express the same constraints.
- **`Promise<T>`** denotes asynchronous return; `Stream<T>` denotes a streaming response that yields zero-or-more items in order.
- **Optional parameters** use `?` after the name; optional fields use `?` after the field name.
- **Errors** are declared per method as the **`(activity, end_state)` pairs** the method can raise — e.g. `transitioning` — `invalid_transition`. Both vocabularies are closed; the registry — enumerated in [Appendix A § Activity end-states](../spec/appendix-a-registries.md#activity-end-states), with semantics in [10 — Error Model § Activity end-states](../spec/10-error-model.md#activity-end-states) — is the single normative home for every pair's meaning and classification (`fix_domain`, `recoverability`, derived `severity`), so per-method declarations never restate classifications. The universal end-states ([Appendix A § Universal end-states](../spec/appendix-a-registries.md#universal-end-states)) apply to every method implicitly and are not repeated per method. Errors are raised as the three-tier chain shaped in [`00-shared-types.md`](./00-shared-types.md#errors--the-three-tier-model); implementations map the chain to whatever error mechanism is idiomatic for their language.
- **Access** is declared per method as the single operation type — from the Tier-1 `OperationKind` vocabulary — that the perimeter's grant check uses; the model is normative in [02 — Conformance § Access control](../spec/02-conformance.md#access-control). The two `authorizing` failure pairs are blanket and never repeated per method. Two Orchestration methods are declared session-floor (callable by any established session, no grant required).
- **Idempotency** contracts are declared once for all methods, in the registry in [`00-shared-types.md § Idempotency`](./00-shared-types.md#idempotency); the defining vocabulary — **idempotent** vs **last-write-wins**, two distinct guarantees — is [01 — Conventions § Idempotency and last-write-wins](../spec/01-conventions.md#idempotency-and-last-write-wins). A method's own **Idempotency** line restates its registry row for reading convenience; where they disagree, the registry governs. Mutating methods with no registry row carry no idempotency contract; read-only methods are naturally idempotent.
- **`changes_since(ref)`** and **`as_stream(ref?)`** are implemented uniformly across stateful containers; their signatures live in `00-shared-types.md` and are cross-referenced, not repeated. `changes_since` yields the current working snapshot's own deltas; `as_stream` yields its full state from empty (cold start), paging and resuming through the same `next_ref` idiom.
- **Paged collection reads** — every unbounded collection read takes `page?: PageRequest` as its final parameter and returns `Page<T>` (items + opaque `next_cursor`), mirroring the Delta-Stream `ChangesPage` idiom; the shared envelope, ordering and consistency rules, and the cursor-defect error mapping live in [`00-shared-types.md § Paged collection reads`](./00-shared-types.md#paged-collection-reads). Collection reads without a `page` parameter are bounded by design.
- **Cross-container calls** — every method that crosses a container boundary specifies the receiving container; intra-container component-to-component calls are implementation-internal and not part of this contract surface.

## What's NOT here

- **Wire formats** (JSON / YAML / protobuf / binary). The interface contract describes operations and types, not their on-the-wire serialization. Serialization and transport bindings are the wire profiles in [13 — Wire Profiles](../spec/13-wire-profiles.md); implementations declare which profiles they support.
- **Implementation-specific options** beyond what the conceptual contract requires.
- **Internal component contracts.** Cross-container public surface only.
- **Sequencing across containers** — cross-container ordering semantics are owned by the spec chapters ([04](../spec/04-snapshots.md), [05](../spec/05-delta-streams.md), [07](../spec/07-atom-lifecycle.md)).
