# 01 — Conventions

## Normative keywords

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt). Briefly:

| Keyword | Meaning |
|---|---|
| MUST / SHALL / REQUIRED | Absolute requirement. A conformant implementation MUST satisfy. |
| MUST NOT / SHALL NOT | Absolute prohibition. |
| SHOULD / RECOMMENDED | There may be valid reasons to deviate, but the full implications must be understood. |
| SHOULD NOT / NOT RECOMMENDED | There may be valid reasons to permit, but the full implications must be understood. |
| MAY / OPTIONAL | Truly optional. An implementation that includes the option is conformant; one that omits it is conformant. |

Normative keywords appear in **bold uppercase**. Lowercase use of the same word (e.g., "must be careful") carries no normative weight.

## Terminology

| Term | Definition |
|---|---|
| **Atom** | A structured claim. The unit of canonical knowledge. See [03 — Data Model](./03-data-model.md). |
| **Snapshot** | An addressable, coherent version of all snapshot-bound state. See [04 — Snapshots](./04-snapshots.md). |
| **Container** | A capability unit at the L2 level. See [`docs/L2/`](../L2/). |
| **Component** | A unit within a container. Informative only; not part of the binding contract. See [`docs/L3/`](../L3/). |
| **Use-case key** | A registered identifier (e.g., `aala.extraction`) that the LLM Gateway routes to a model. See [09 — LLM Gateway](./09-llm-gateway.md). |
| **Delta stream** | The ordered event log a container exposes via `changes_since(ref)`. See [05 — Delta Streams](./05-delta-streams.md). |
| **Ref** | An opaque checkpoint identifier used to position a consumer in a delta stream. |
| **Present state** | An atom status the atom can be in (`Active`, `Hanging`, `Deferred`, `Deprecated`). |
| **Removal outcome** | An atom transition that takes the atom out of the snapshot (`Superseded`, `Duplicate`, `Rejected`, `Removed`). |
| **Conformant implementation** | An implementation that satisfies every MUST in this spec. See [02 — Conformance](./02-conformance.md). |
| **Implementer** | The party building an aala implementation. |
| **Deployer** | The party running an aala instance. |
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

## Cross-references

Spec chapters reference [`docs/interfaces/`](../interfaces/) for binding method signatures. The spec adds the cross-cutting semantics — behavior, invariants, ordering, retry, conformance — that the interface files leave implicit.

Spec chapters reference [`docs/L2/`](../L2/) for the container partition and concern descriptions. L2 narrative is informative; the partition itself (the 9 containers) is normative.

[`docs/L3/`](../L3/) is informative only — it describes one valid internal organization for each container. An implementation MAY use a different internal organization.
