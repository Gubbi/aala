# 11 — Versioning

This chapter defines how aala evolves: spec version identifiers, which changes are backward-compatible and which are breaking, and how implementations and callers negotiate versions.

## Version identifiers

Each container's interface declares a version string in its interface file (e.g., `aala-v0.1`). The spec as a whole has a corresponding version that aggregates all container interface versions plus the cross-cutting rules.

Format: `aala-v<major>` or `aala-v<major>.<minor>`. The `<major>` portion increments for breaking changes; `<minor>` increments for backward-compatible additions. The scheme has these two components only — there is no patch position (in [SemVer](https://semver.org) terms, the `<minor>` component plays the *minor* role, and backward-compatible bug fixes ride it like any other backward-compatible change).

Pre-publish versions use the `v0.<minor>` series. In-place overwrite of the spec is permitted within `v0.x` — no migration story required. The first published major version is `v1`.

Examples: `aala-v0.1`, `aala-v0.2`, `aala-v1`, `aala-v1.1`, `aala-v2`.

## Backward-compatible changes

A change is **backward-compatible** if every conformant implementation of the prior version remains conformant. Specifically, the following are backward-compatible:

- Adding a new optional field to a data type (e.g., new optional fields on `Atom`).
- Adding a new ClassificationAtom to the standard library — implementations preserve existing classifications and add the new one.
- Adding a new PredicateAtom to the standard library.
- Adding a new event kind to a Delta Stream — consumers MUST tolerate unknown event kinds per [05](./05-delta-streams.md).
- Adding a new optional method to a container interface.
- Adding a new optional method parameter.
- Adding a new registered LLM Gateway use-case key (the key is opt-in; existing implementations without the key remain conformant if they don't use it).
- Adding a new optional container (per [02](./02-conformance.md), the optional container set MAY grow).
- Adding a new `end_state` to an activity's registry in [10 — Error Model § Activity end-states](./10-error-model.md#activity-end-states) — the registry entry fixes the new pair's classification, and consumers MUST tolerate unknown `end_state` values by falling back to the (closed) `recoverability` / `fix_domain` classification, which fully determines control flow. The `Activity`, `OperationKind`, `FixDomain`, `Recoverability`, and `Severity` vocabularies are closed; growing them is a breaking change.
- Adding a new optional attribute in a ClassificationSchema (existing classified atoms remain valid).
- Tightening behavior to fix bugs, where the new behavior is a strict refinement of the old (every call that succeeded previously still succeeds, and now produces a more accurate result).

## Breaking changes

A change is **breaking** if it MAY cause a previously conformant implementation to become non-conformant. The following are breaking:

- Adding or removing a concrete `AtomType` value. The set of five concrete types (`entity`, `relation`, `classification`, `predicate`, `predicate_kind`) is closed.
- Adding or removing a `PredicateKind` value. The ten values are closed and carry hard semantics; adding a kind requires every implementation to handle its cascade rules.
- Adding or removing an `Activity`, `OperationKind`, `FixDomain`, or `Recoverability` value, or changing the `Severity` derivation. These error-model vocabularies are closed — they are the cross-vendor contract (see [10 — Error Model](./10-error-model.md)).
- Removing a field, method, parameter, or enum variant.
- Repurposing an existing field, method, or variant to mean something different.
- Tightening an idempotency rule (e.g., changing a method from idempotent to non-idempotent).
- Loosening an idempotency rule (callers may have relied on the strict version).
- Changing concurrency semantics (e.g., changing from "serialized per snapshot" to "concurrent allowed").
- Changing the meaning of a removal outcome.
- Changing the cascade rules of a predicate kind or the schema-inheritance rules.
- Removing or repurposing a registered LLM Gateway use-case key.
- Changing the required arguments of an existing method.
- Changing what makes a container's container realization conformant (e.g., promoting an optional container to required).
- Removing a standard-library ClassificationAtom or PredicateAtom.

Breaking changes MUST bump the major version (once past `v0.x`).

## Version negotiation

A caller can query an implementation's version per container (`OrchestrationCapabilities` shape in [`interfaces/orchestration.md` § `capabilities`](../interfaces/orchestration.md#capabilities)):

```typescript
const caps = await orchestration.capabilities()
// caps.capabilities: [{ id: "atoms", version: "aala-v0.1", ... }, ...]
```

A caller written against `aala-v0.1` MAY assume every container declaring `aala-v0.1` honors the contract as of `aala-v0.1`. A container declaring `aala-v0.2` honors `aala-v0.1` plus the backward-compatible additions documented for `aala-v0.2`.

Callers SHOULD check capability versions at startup and behave accordingly. A caller written for `aala-v1` MUST NOT assume an earlier implementation supports v1 additions.

## Minor increments

When a container adds backward-compatible features:

1. The container's version string SHOULD bump (e.g., `aala-v0.1` → `aala-v0.2`). This is informational; conformance to the prior version is unaffected.
2. The interface file MUST document the additions, with a "Version notes" entry indicating the minor version they appeared in.
3. The spec MAY bump its version to match.

## Major increments

When a breaking change is required (post-`v0.x`):

1. A new spec major version MUST be released (e.g., `aala-v2`).
2. The interface files MUST document the breaking changes explicitly.
3. The previous major version MAY remain supported by implementations as a separate declared capability. Implementations MAY declare multiple major versions in their `capabilities()` if they support both.

## Deprecation

Within a major version, features MAY be marked **deprecated** without being removed. Deprecated features:

- MUST continue to function exactly as documented within the current major version.
- SHOULD emit a warning (logged, observable, or in API metadata) on use.
- MAY be removed in the next major version.

Implementations MUST document which features they consider deprecated.

## Schema evolution rules

For the atom data model specifically:

- **Adding a new atom field**: backward-compatible if the field is optional. Atoms without the field MUST remain valid.
- **Adding a new `AtomType`**: breaking. The five concrete types are closed.
- **Adding a new `PredicateKind`**: breaking. The ten kinds are closed.
- **Adding a new ClassificationAtom to the standard library**: backward-compatible.
- **Adding a new PredicateAtom (standard or deployment-specific)**: backward-compatible.
- **Adding a new `AtomStatus` value**: breaking by default, because consumers may depend on the closed set of statuses. A status addition is backward-compatible only when it was declared in advance (in the published `aala-v0.X` spec) so consumers know to handle it.
- **Adding attributes to a ClassificationSchema**: backward-compatible if optional; breaking if required (would invalidate existing classified atoms missing the field).
- **Adding `required_relations` to a ClassificationSchema on the existing class**: breaking-by-effect (existing classified atoms transition to hanging) — treat as a major bump even if syntactically the field is just an additional list entry.
- **Renaming a field**: always breaking.
- **Changing a field's type**: always breaking.

## Cross-version implementations

An implementation MAY support multiple versions concurrently (e.g., declaring `aala-v1` and `aala-v2`). In this case:

- The implementation MUST honor the contract of each declared version fully.
- The implementation MUST allow callers to select a version (via API field, header, or routing).
- The implementation SHOULD document its multi-version support in `capabilities()` extension fields.

## Forward compatibility

Consumers SHOULD be tolerant of backward-compatible additions:

- Unknown event kinds in Delta Streams: treat as no-ops.
- Unknown enum variants where forward-compatibility is declared: handle gracefully (e.g., log + fallback path).
- Unknown extra fields on data structures: ignore.
- Unknown ClassificationAtoms / PredicateAtoms in the standard library: pass through; treat as opaque.

This tolerance is the consumer's responsibility, not the producer's.
