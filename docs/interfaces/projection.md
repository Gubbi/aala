# Projection — Interface Contract

**Version:** `aala-v1`
**Optionality:** non-optional (for human-reviewed deployments)
**Container:** [`L2/04-projection.md`](../L2/04-projection.md) · [`L3/04-projection.md`](../L3/04-projection.md)

Projection renders atoms into human-readable prose in the current snapshot. It owns the rendered files, the claim-to-section reverse index, and the render cache.

## Methods

### `re_render`

```typescript
function re_render(scope?: ScopePath): Promise<ReRenderResult>

interface ReRenderResult {
  changed_sections:   SectionPath[]
  driving_atoms:      AtomId[]   // atoms that drove the changed sections
  cache_hits:         number
  cache_misses:       number
}
```

Produces or updates projection files for the affected sections in the current snapshot. With no scope, re-renders everything implied by atoms that changed since the last render. With a scope, re-renders that scope's sections regardless of the last-rendered ref.

In steady-state operation, Projection's Delta Consumer subscribes to Atoms's change stream and triggers re-renders incrementally; explicit `re_render` is for initial population, full rebuilds, or scoped recomputation.

**Errors:**
```typescript
type ReRenderError =
  | CommonError
  | { kind: "RenderFailed"; section: SectionPath; reason: string }
```

**Idempotency:** re-rendering with the same atom set produces byte-identical output (determinism property). Repeated calls without intervening atom changes are no-ops at the byte level but still return a `ReRenderResult`.

**Concurrency:** serialized per snapshot.

---

### `get`

```typescript
function get(section: SectionPath): Promise<string | null>
```

Fetch a rendered section from the current snapshot. Returns `null` if not present.

**Errors:** `CommonError`.

---

### `affected_sections`

```typescript
function affected_sections(atom_ids: AtomId[]): Promise<SectionPath[]>
```

Which projection files / sections would change if these atoms changed. Lets callers preview render scope without doing the work. Pure read against the Reverse Index.

**Errors:** `CommonError`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type ProjectionEvent =
  | { kind: "SectionChanged";  payload: { section: SectionPath; driving_atoms: AtomId[] } }
  | { kind: "SectionAdded";    payload: { section: SectionPath; driving_atoms: AtomId[] } }
  | { kind: "SectionDeleted";  payload: { section: SectionPath; reason: string } }
  | { kind: "IndexUpdated";    payload: { section: SectionPath; atoms_added: AtomId[]; atoms_removed: AtomId[] } }
```

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible: new event kinds, new optional fields in `ReRenderResult`.

## Notes for implementers

- The render cache is internal to Projection. Cache hashing semantics (`atom_set + prompt_version + model_version`) are implementation choice; the contract is "given the same atoms, the same prompt template version, and the same model, the rendered output is byte-identical."
- The narrative renderer calls LLM Gateway with `aala.projection_narrative`. Template-only configurations skip this call.
- Anchoring (passing previous prose to the narrative renderer) is a quality property: minimizes gratuitous rephrasing on re-render. Required for deterministic diffs.
