# Atoms — Interface Contract

**Version:** `aala-v1`
**Optionality:** non-optional (every aala deployment exposes Atoms)
**Container:** [`L2/03-atoms.md`](../L2/03-atoms.md) · [`L3/03-atoms.md`](../L3/03-atoms.md)

Atoms is the canonical record. It owns the atom schema, the durable store, extraction (Create), conflict detection, status transitions, alias and reference primitives, and the embedding-NN index used internally by Conflict.

## Methods

### `propose`

```typescript
function propose(
  fragment:         NormalizedFragment,
  extractor_hints?: ExtractorHints
): Promise<ProposeResult>
```

Runs extraction + conflict detection. Mutates the current snapshot atomically: accepted atoms are added; affected atoms (per Conflict's classification) are mutated through Status Manager; Hanging cascade via direct references is applied.

```typescript
interface ExtractorHints {
  prefer?:          string[]    // extractor ids to prefer
  exclude?:         string[]    // extractor ids to skip
  content_kind?:    ContentKind // override the fragment's content_kind for routing
}

interface ProposeResult {
  ingest_id:        IngestId
  added:            AtomId[]
  classifications:  AtomClassification[]
  cascaded_hanging: AtomId[]   // atoms transitioned to Hanging by direct-ref cascade
}

interface AtomClassification {
  atom_id:          AtomId      // the new atom (for accept outcomes) or the affected canonical atom (for Superseded / Refined / etc.)
  outcome:          ConflictOutcome
  rationale:        string
  candidates:       AtomId[]    // canonical candidates the judge considered
  meta:             Record<string, unknown>  // e.g., superseded_by, duplicate_of
}

type ConflictOutcome =
  | "New"
  | "Compatible"
  | "Duplicate"
  | "Reinforces"
  | "Refines"
  | "Supersedes"
  | "Contradicts"
  | "Unclear"
```

**Errors:**
```typescript
type ProposeError =
  | CommonError
  | { kind: "ExtractionFailed"; fragment_id?: FragmentId; reason: string }
  | { kind: "SchemaViolation"; atom_id: AtomId; reason: string }
```

**Idempotency:** idempotent on `fragment.message_id`. Re-proposing the same message_id within the same snapshot is a no-op; the original `ProposeResult` may be returned cached.

**Concurrency:** serialized per snapshot.

---

### `update_status`

```typescript
function update_status(
  atom_id:    AtomId,
  outcome:    StatusOutcome,
  rationale:  string,
  meta?:      StatusOutcomeMeta
): Promise<void>
```

Applies a present-state transition or a removal outcome to an atom. All such mutations go through Atoms's Status Manager (the only mutator of canonical atom state).

```typescript
type StatusOutcome =
  | "Active"          // present state
  | "Hanging"         // present state
  | "Deferred"        // present state
  | "Deprecated"      // present state
  | "Superseded"      // removal outcome (meta.superseded_by required)
  | "Duplicate"       // removal outcome (meta.duplicate_of required)
  | "Rejected"        // removal outcome
  | "Removed"         // removal outcome (generic)

interface StatusOutcomeMeta {
  superseded_by?:     AtomId
  duplicate_of?:      AtomId
  blast_id?:          BlastId         // when triggered by a Blast Radius resolution
  idempotency_key?:   string          // optional caller-supplied key
}
```

**Errors:**
```typescript
type UpdateStatusError =
  | CommonError
  | { kind: "InvalidTransition"; from: AtomStatus; to: StatusOutcome; reason: string }
  | { kind: "MissingMeta"; required: keyof StatusOutcomeMeta }
```

`Superseded` and `Duplicate` require the corresponding meta field; absence is `MissingMeta`.

**Idempotency:** idempotent on `(atom_id, outcome, meta.idempotency_key?)`. Last write wins on `status_changed_at`. Calling with the same `(atom_id, outcome)` and equivalent meta is a no-op.

**Concurrency:** serialized per atom within a snapshot.

---

### `get_by_id`

```typescript
function get_by_id(atom_id: AtomId): Promise<Atom | null>
```

Returns the atom in the current snapshot, or `null` if not present. Atoms in removal-outcome states (`Superseded` / `Duplicate` / `Rejected` / `Removed`) are returned only by implementations that persist them as tombstones; impls using actual deletion return `null`.

**Errors:** `CommonError`.

**Concurrency:** concurrent-safe.

---

### `list_scope`

```typescript
function list_scope(
  scope_path:      ScopePath,
  status_filter?:  AtomStatus[]
): Promise<Atom[]>
```

Atoms at one node of the atom tree. `status_filter` defaults to present states only (`["active", "hanging", "deferred", "deprecated"]`); pass the full set to include tombstones (impl-dependent).

**Errors:** `CommonError | { kind: "InvalidScope"; scope_path: string }`.

---

### `list_children`

```typescript
function list_children(scope_path: ScopePath): Promise<ScopePath[]>
```

Child scopes of a tree node.

**Errors:** `CommonError | { kind: "InvalidScope"; scope_path: string }`.

---

### `traverse_references`

```typescript
function traverse_references(
  atom_id:    AtomId,
  direction:  "outgoing" | "incoming"
): Promise<AtomId[]>
```

Walks the reference graph from one atom. `outgoing` returns atoms this atom points at (its `references` + `depends_on`); `incoming` returns atoms pointing at it (reverse direction).

**Errors:** `CommonError`.

---

### `changes_since`

Conforms to the shared signature from [`00-shared-types.md`](./00-shared-types.md#change-stream-events). Event variants:

```typescript
type AtomsEvent =
  | { kind: "Added";          payload: { atom_id: AtomId; atom: Atom } }
  | { kind: "Updated";        payload: { atom_id: AtomId; before: Atom; after: Atom } }
  | { kind: "StatusChanged";  payload: { atom_id: AtomId; from: AtomStatus; to: AtomStatus; rationale: string } }
  | { kind: "Superseded";     payload: { atom_id: AtomId; superseded_by: AtomId; rationale: string } }
  | { kind: "Duplicate";      payload: { atom_id: AtomId; duplicate_of: AtomId; rationale: string } }
  | { kind: "Rejected";       payload: { atom_id: AtomId; rationale: string } }
  | { kind: "Removed";        payload: { atom_id: AtomId; rationale: string } }
```

**Ordering:** events within the stream are totally ordered. The combination of `snapshot(ref_X) + events(ref_X → ref_Y)` is exactly `snapshot(ref_Y)`.

---

## Version notes

- **`aala-v1`** — initial contract.
- Patch-compatible additions allowed: new optional fields in `Atom`, new `AtomType` variants, new `ConflictOutcome` variants, new `StatusOutcome` variants, new event kinds (consumers must tolerate unknown event kinds as no-ops).
- Breaking changes (bump major): removing or repurposing existing variants; changing required parameters; changing idempotency rules.

## Notes for implementers

- Status Manager is the only path that mutates the Atom Store. Conflict's classification work, the Hanging cascade, and the `update_status` call all funnel through it. This is enforced by the [API-only access principle](../L2/01-principles.md).
- The Embedding Index used by Conflict is not part of the public interface. Implementations may use any NN technique (or none — see [`L3/03-atoms.md`](../L3/03-atoms.md) variation points).
- Removal outcomes (`Superseded` / `Duplicate` / `Rejected` / `Removed`) may be applied as actual deletion or as tombstone statuses. Either is conformant; the conceptual model treats them as intent-carrying outcomes regardless of persistence.
