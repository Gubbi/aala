# 02 — Conformance

This chapter defines what makes an implementation **conformant**. The remaining spec chapters describe the rules; this chapter defines what an implementation must satisfy and how it declares its capabilities.

## Required containers

A conformant implementation MUST realize the following containers:

| Container | Status |
|---|---|
| [Ingestion](../interfaces/ingestion.md) | REQUIRED |
| [Atoms](../interfaces/atoms.md) | REQUIRED |
| [Projection](../interfaces/projection.md) | REQUIRED |
| [Orchestration](../interfaces/orchestration.md) | REQUIRED |
| [LLM Gateway](../interfaces/llm-gateway.md) | REQUIRED *unless* the implementation performs zero LLM calls (no extraction, no conflict judge, no narrative rendering, no synthesis, no LLM-as-judge) |

A conformant implementation MAY realize the following containers. When realized, the corresponding interface contract MUST be honored in full:

| Container | Status |
|---|---|
| [Hierarchical Navigation](../interfaces/hierarchical-nav.md) | OPTIONAL |
| [Blast Radius](../interfaces/blast-radius.md) | OPTIONAL |
| [Synthesis](../interfaces/synthesis.md) | OPTIONAL |
| [Quality](../interfaces/quality.md) | OPTIONAL |

## Capability declaration

Every conformant implementation MUST expose `Orchestration.capabilities()`. The return value MUST list every container the implementation realizes, with its declared version. The list MUST be stable for the lifetime of the implementation process unless the deployer reconfigures it.

```typescript
[
  { id: "ingestion",        version: "aala-v1", optional: false, wired: true  },
  { id: "atoms",            version: "aala-v1", optional: false, wired: true  },
  { id: "projection",       version: "aala-v1", optional: false, wired: true  },
  { id: "orchestration",    version: "aala-v1", optional: false, wired: true  },
  { id: "llm_gateway",      version: "aala-v1", optional: false, wired: true  },
  { id: "hierarchical_nav", version: "aala-v1", optional: true,  wired: false },
  { id: "blast_radius",     version: "aala-v1", optional: true,  wired: true  },
  { id: "synthesis",        version: "aala-v1", optional: true,  wired: true  },
  { id: "quality",          version: "aala-v1", optional: true,  wired: false }
]
```

A caller MAY consult this declaration to decide which features to invoke.

## Interface conformance

For each realized container, the implementation:

- **MUST** expose every method defined in the corresponding interface file under [`docs/interfaces/`](../interfaces/).
- **MUST** honor every signature exactly: parameter types, return types, error variants.
- **MUST** honor the declared idempotency rules.
- **MUST** honor the declared concurrency model (serialized writes, concurrent-safe reads).
- **MUST** emit every event variant defined in the container's `changes_since` section.
- **MAY** expose additional methods, additional fields, or additional event variants. Such extensions MUST NOT change the meaning of existing methods or events.
- **MAY** expose additional registered LLM Gateway use-case keys beyond the 9 standard ones (see [09 — LLM Gateway](./09-llm-gateway.md)). All standard keys MUST be present.

## Cross-cutting invariants

A conformant implementation MUST honor the cross-cutting rules in the remaining spec chapters:

- **Data Model** ([03](./03-data-model.md)): atom shape, types, statuses.
- **Snapshots** ([04](./04-snapshots.md)): lifecycle, coherence, isolation.
- **Delta Streams** ([05](./05-delta-streams.md)): ordering, replay equivalence.
- **Conflict Classification** ([06](./06-conflict-classification.md)): outcome semantics.
- **Atom Lifecycle** ([07](./07-atom-lifecycle.md)): transitions and their triggers.
- **Blast Radius** ([08](./08-blast-radius.md)) when wired: pipeline stages, iteration policy.
- **LLM Gateway** ([09](./09-llm-gateway.md)) when wired: 9 standard use-case keys.
- **Error Model** ([10](./10-error-model.md)): retry semantics, idempotency guarantees.
- **Versioning** ([11](./11-versioning.md)): patch-compatible additions, breaking-change rules.
- **Edge Cases** ([12](./12-edge-cases.md)): concurrent operations, malformed inputs, convergence.

## API-only access

Every interaction with a container's state MUST go through the container's published interface. An implementation MUST NOT permit external entities (including other containers) to read or mutate a container's internal state directly. This applies at every level — between containers, and between components inside a container.

## Conformance test posture

A reference conformance test suite MAY be provided alongside the spec; running it is RECOMMENDED but not required. The binding test is "does the implementation satisfy every MUST in this spec." The test suite, when present, exercises a representative subset.
