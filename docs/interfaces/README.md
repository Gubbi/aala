# Interface Contracts

Formal interface contracts for each L2 container. These are the binding agreements between containers — anything you can call across a container boundary in aala lives here, with exact signatures, error model, idempotency rules, and concurrency semantics.

## Audience

- **Implementer** writing a concrete realization of a container (or all of aala).
- **Spec author** drawing on these to assemble `docs/spec/` — the consolidated formal specification.
- **Reader** comparing implementations or building a client.

## What's here

| File | Covers |
|---|---|
| [`00-shared-types.md`](./00-shared-types.md) | Identifiers, common data types, error variants, event types — shared across multiple containers. Read this first. |
| [`ingestion.md`](./ingestion.md) | Ingestion (non-optional) |
| [`atoms.md`](./atoms.md) | Atoms (non-optional). The central interface; many others reference it. |
| [`projection.md`](./projection.md) | Projection (non-optional) |
| [`orchestration.md`](./orchestration.md) | Orchestration (non-optional) — the outside-world entry point |
| [`hierarchical-nav.md`](./hierarchical-nav.md) | Hierarchical Navigation (optional) |
| [`blast-radius.md`](./blast-radius.md) | Blast Radius (optional) |
| [`synthesis.md`](./synthesis.md) | Synthesis (optional) |
| [`llm-gateway.md`](./llm-gateway.md) | LLM Gateway (cross-cutting infrastructure) |
| [`quality.md`](./quality.md) | Quality (optional cross-cutting measurement) |

## Conventions

- **Signature syntax** is TypeScript-like for readability. The shape is what's binding, not the syntax — implementations may use any language whose types can express the same constraints.
- **`Promise<T>`** denotes asynchronous return; `Stream<T>` denotes a streaming response that yields zero-or-more items in order.
- **Optional parameters** use `?` after the name; optional fields use `?` after the field name.
- **Errors** are enumerated per method as a union type. Implementations should map these to whatever error mechanism is idiomatic for their language.
- **`changes_since(ref)`** is implemented uniformly across stateful containers; its signature lives in `00-shared-types.md` and is cross-referenced, not repeated.
- **Cross-container calls** — every method that crosses a container boundary specifies the receiving container; intra-container component-to-component calls are documented in `docs/L3/` and are not part of this contract surface.

## What's NOT here

- **Wire formats** (JSON / YAML / protobuf / binary). The interface contract describes operations and types, not their on-the-wire serialization. Implementations make wire choices in `docs/implementations/<impl>/`.
- **Implementation-specific options** beyond what the conceptual contract requires.
- **Internal component contracts.** Cross-container public surface only.
- **Sequencing across containers** — that's in [`L2/11-flows.md`](../L2/11-flows.md).
