# 11 — Versioning

This chapter defines how aala evolves: spec version identifiers, what changes are patch-compatible vs breaking, and how implementations and callers negotiate versions.

## Version identifiers

Each container's interface declares a version string in its interface file (e.g., `aala-v1`). The spec as a whole has a corresponding version that aggregates all container interface versions plus the cross-cutting rules.

Format: `aala-v<major>` or `aala-v<major>.<minor>`. The `<major>` portion increments for breaking changes; `<minor>` increments for patch-compatible additions.

Examples: `aala-v1`, `aala-v1.1`, `aala-v2`.

## Patch-compatible changes

A change is **patch-compatible** if every conformant implementation of the prior version remains conformant. Specifically, the following are patch-compatible:

- Adding a new optional field to a data type (e.g., new optional fields on `Atom`).
- Adding a new enum variant (e.g., a new `AtomType` value) — provided that consumers were already specified to tolerate unknown variants.
- Adding a new event kind to a delta stream — consumers MUST tolerate unknown event kinds per [05](./05-delta-streams.md).
- Adding a new optional method to a container interface.
- Adding a new optional method parameter.
- Adding a new registered LLM Gateway use-case key (the key is opt-in; existing impls without the key remain conformant if they don't use it).
- Adding a new optional container (per [02](./02-conformance.md), the optional container set MAY grow).
- Adding new error variants — consumers MUST be prepared for unknown variants (typically treated as `InternalError`).
- Tightening behavior to fix bugs, where the new behavior is a strict refinement of the old (every call that succeeded previously still succeeds, and now produces a more accurate result).

## Breaking changes

A change is **breaking** if it MAY cause a previously conformant implementation to become non-conformant. The following are breaking:

- Removing a field, method, parameter, or enum variant.
- Repurposing an existing field, method, or variant to mean something different.
- Tightening an idempotency rule (e.g., changing a method from idempotent to non-idempotent).
- Loosening an idempotency rule (callers may have relied on the strict version).
- Changing concurrency semantics (e.g., changing from "serialized per snapshot" to "concurrent allowed").
- Changing the meaning of a removal outcome (e.g., redefining `Superseded` to imply something different).
- Removing or repurposing a registered LLM Gateway use-case key.
- Changing the required arguments of an existing method.
- Changing what makes a container's container realization conformant (e.g., promoting an optional container to required).

Breaking changes MUST bump the major version.

## Version negotiation

A caller can query an implementation's version per container:

```typescript
const caps = await orchestration.capabilities()
// caps: [{ id: "atoms", version: "aala-v1", ... }, ...]
```

A caller written against `aala-v1` MAY assume every container declaring `aala-v1` honors the contract as of `aala-v1`. A container declaring `aala-v1.1` honors `aala-v1` plus the patch additions described in this chapter.

Callers SHOULD check capability versions at startup and behave accordingly. A caller written for `aala-v2` MUST NOT assume an `aala-v1` implementation supports v2 additions.

## Patch increments

When a container adds patch-compatible features:

1. The container's version string SHOULD bump (e.g., `aala-v1` → `aala-v1.1`). This is informational; conformance to `aala-v1` is unaffected.
2. The interface file MUST document the additions, with a "Version notes" entry indicating the patch level they appeared in.
3. The spec MAY bump its version to match.

## Major increments

When a breaking change is required:

1. A new spec major version MUST be released (e.g., `aala-v2`).
2. The interface files MUST document the breaking changes explicitly.
3. The previous major version (`aala-v1`) MAY remain supported by implementations as a separate declared capability. Implementations MAY declare multiple major versions in their `capabilities()` if they support both.

## Deprecation

Within a major version, features MAY be marked **deprecated** without being removed. Deprecated features:

- MUST continue to function exactly as documented within the current major version.
- SHOULD emit a warning (logged, observable, or in API metadata) on use.
- MAY be removed in the next major version.

Implementations MUST document which features they consider deprecated.

## Schema evolution rules

For the atom data model specifically:

- **Adding a new atom field**: patch-compatible if the field is optional. Atoms without the field MUST remain valid.
- **Adding a new `AtomType`**: patch-compatible. Implementations not recognizing the type MUST treat it as a generic atom (subject/predicate/object accessible; type-specific extractors and validations skipped).
- **Adding a new `AtomStatus` value**: NORMALLY breaking, because consumers may depend on the closed set of statuses. To make a status addition patch-compatible, the addition MUST be declared in advance (in the published `aala-v1.X` spec) so consumers know to handle it.
- **Renaming a field**: always breaking.
- **Changing a field's type**: always breaking.

## Cross-version implementations

An implementation MAY support multiple versions concurrently (e.g., declaring `aala-v1` and `aala-v2`). In this case:

- The implementation MUST honor the contract of each declared version fully.
- The implementation MUST allow callers to select a version (via API field, header, or routing).
- The implementation SHOULD document its multi-version support in `capabilities()` extension fields.

## Forward compatibility

Consumers SHOULD be tolerant of patch-compatible additions:

- Unknown event kinds in delta streams: treat as no-ops.
- Unknown enum variants: handle gracefully (e.g., log + fallback path).
- Unknown extra fields on data structures: ignore.

This tolerance is the consumer's responsibility, not the producer's.
