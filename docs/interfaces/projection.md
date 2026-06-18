# Projection — Facet Interface Contract

**Version:** `aala-v0.1`
**Kind:** cross-cutting **read facet**, not a container.
**Implemented by:** every *contentful* container, over its own content (required of Atoms — see [02 — Conformance](../spec/02-conformance.md)).

Projection is the uniform, read-only surface through which a contentful container exposes a human- and LLM-readable **prose rendering** of its own state, plus a hierarchical index for navigating it. It is a *facet* — a small interface every contentful container implements — not an independent service that owns canonical state. The rendering serves three intents at once: **human review** (the diff a reviewer reads), **LLM read/grounding** (the prose an external agent grounds on), and **export/interchange** (a portable bundle other organizations and tools can consume).

Projection is a **derived, change-log-driven materialized read model** — like Blast Radius and Hierarchical Navigation. The implementing container consumes its *own* source content's Delta Stream and keeps a materialized projection store in sync; the read methods below serve that store. **Nothing renders per call** — a `read` returns a document the container already materialized, never one synthesized on the request path. This is why there is no `re_render` method (see [§ Notes for implementers](#notes-for-implementers)).

A container that implements this facet renders its own content to markdown and owns the artifacts behind it (the materialized projection store, reverse index, render cache, and — for Atoms — the glossary). Uniform read access across all projections is a property of this shared interface plus the perimeter's grant check ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)): every facet method declares **Access:** `query`, so one grant class covers every projection — no central service is involved.

## Output profile

The materialized store conforms to a **single output profile** configured at deploy time. The profile fixes the rendered document structure, the aala→document content mapping, and the index/log serialization — the **profile contract** is enumerated in [Appendix A § Projection output profiles](../spec/appendix-a-registries.md#projection-output-profiles). The default profile is [`okf`](../spec/appendix-b-okf-profile.md) (Open Knowledge Format — a cross-organization interchange format, the natural fit for the export intent). A deployment **MAY** substitute exactly **one** custom profile that satisfies the profile contract.

A deployment runs **exactly one** profile. There is no `profile?` argument on any read method, no concurrent profile stores, and no per-call profile selection: the read surface serves whichever profile the materialized store was built under, advertised once via [`OrchestrationCapabilities.projection_profile`](./orchestration.md#capabilities). Changing the configured profile is an operational re-projection, not a runtime choice (see [§ Notes for implementers](#notes-for-implementers)).

`ProfileKey` is a path-style name (`/`-free, namespaced like `UseCaseKey`); the built-in default is `okf`.

## Who implements it

| Container | Obligation | Projects |
|---|---|---|
| [Atoms](./atoms.md) | **MUST** | Canonical claim prose + glossary — the surface humans review. |
| [Ingestion](./ingestion.md) | MAY | Normalized raw fragments as readable source documents. |
| [Blast Radius](./blast-radius.md) | MAY | Blast reports as readable impact narratives. |
| [Hierarchical Navigation](./hierarchical-nav.md) | MAY | Axis trees as navigable outlines. |

The facet is a **separate read surface** from a container's structured read API. The structured API answers precise graph queries (`get_by_id`, `traverse_relations`, …); the projection facet answers "give me readable prose I can ground an LLM on, and let me navigate it." See [`docs/analysis/agent-integration-pattern.md`](../analysis/agent-integration-pattern.md) for how an external agent uses the two together.

## Path type

Projection paths reuse `SectionPath` (path-shaped, `/`-separated, filesystem-friendly; an optional `#fragment` addresses a section within a file). Example: `domains/payments/services/checkout.md#flow`.

## Methods

### `list`

```typescript
function list(prefix?: SectionPath, page?: PageRequest): Promise<Page<SectionPath>>
```

Enumerate the projection paths this container currently exposes, optionally filtered to those under `prefix`. Filesystem/S3-like listing — no content is returned. Paths are stable across renders unless the underlying content is added or removed. Paged per [00 — Shared Types § Paged collection reads](./00-shared-types.md#paged-collection-reads).

**Errors:** `querying` — `invalid_query` (malformed prefix, or invalid paging cursor).

**Access:** `query`.

---

### `read`

```typescript
function read(path: SectionPath): Promise<ProjectionDoc | null>

interface ProjectionDoc {
  path:         SectionPath
  content:      string                 // rendered markdown (frontmatter + body)
  frontmatter:  Record<string, unknown> // the profile's structured document head, parsed
  provenance:   AtomId[]               // atoms this document was rendered from
  profile:      ProfileKey             // the output profile this document conforms to
  rendered_ref: Ref                    // the content ref this rendering reflects
}
```

Fetch one rendered projection document from the materialized store. Returns `null` if the path is absent. `content` is the full rendered markdown including its frontmatter; `frontmatter` is the same profile-defined structured head parsed out, so a caller reads document metadata (title, type, timestamp, profile-namespaced keys) without re-parsing the markdown. `profile` names the output profile the document conforms to — the same `ProfileKey` the deployment advertises ([§ Output profile](#output-profile)). `provenance` lets a caller jump from prose back to the structured atoms behind it — hydrated in one round trip with [`Atoms.get_by_ids`](./atoms.md#get_by_ids), not one `get_by_id` per atom.

**Errors:** `querying` — universal end-states only (absence is `null`, not an error).

**Access:** `query`.

---

### `read_index`

```typescript
function read_index(root?: SectionPath, depth?: number): Promise<ProjectionIndex>

interface ProjectionIndex {
  node:      SectionPath               // this node's path ("" / root if omitted)
  title:     string                    // human/LLM-facing label for this node
  summary:   string | null             // one-line gist, for descent decisions without fetching the leaf
  children:  ProjectionIndex[]         // sub-nodes; empty for leaves — and at a `depth` cut (is_leaf disambiguates)
  is_leaf:   boolean                   // true → a `read(node)` returns a document
}
```

Return the hierarchical navigation index (PageIndex-style): a tree of titled, summarized nodes an agent descends to locate the few leaves it needs, fetching full prose only for those. Optionally rooted at `root` for partial-tree fetches; `depth` caps how many levels below the root are expanded (absent = the full subtree). A node at the depth cut returns `children: []` with `is_leaf: false` — the caller continues with `read_index(node)`. Hierarchy-scoped descent (`root` + `depth`), not a cursor, is this read's bounding affordance.

**Errors:** `querying` — `not_found` (unknown `root`), `invalid_query` (`depth` < 0).

**Access:** `query`.

---

### `changes_since`

Shared signature. Event variants:

```typescript
type ProjectionEvent =
  | { kind: "DocChanged";     payload: { path: SectionPath; provenance: AtomId[] } }
  | { kind: "DocAdded";       payload: { path: SectionPath; provenance: AtomId[] } }
  | { kind: "DocRemoved";     payload: { path: SectionPath; reason: string } }
  | { kind: "IndexUpdated";   payload: { node: SectionPath } }
```

The facet's own Delta Stream, so consumers can incrementally track which rendered documents moved.

---

## Determinism

A projection rendering is deterministic on the tuple **(`content_ref`, `profile`, `template_version`, `model`)**: holding the source content ref, the configured output profile, the prompt-template version, and the model fixed, `read(path)` returns byte-identical markdown — frontmatter and body alike. Without this, an unchanged document would re-render to slightly different prose every time and obscure real diffs. The render cache and previous-prose anchoring (implementation concerns) exist to hold this property. The profile is part of the tuple because the same content renders to different documents under different profiles; within a deployment the profile is fixed, so it contributes a constant.

By default, derived atoms (those with `derivation` populated) do not drive rendered documents — projections reflect asserted claims. An implementation MAY expose derived inferences in diagnostic projections behind an explicit opt-in.

## Version notes

- **`aala-v0.1`** — initial facet contract; pre-publish. Default output profile `okf` targets OKF 0.1 ([Appendix B — OKF Profile](../spec/appendix-b-okf-profile.md)).
- Backward-compatible: new event kinds, new optional fields in `ProjectionDoc` / `ProjectionIndex`, new registered output profiles. The output profile and its document format version independently per [Appendix B § Versioning](../spec/appendix-b-okf-profile.md#versioning); a profile-version bump is not an interface-version bump.

## Notes for implementers

- There is **no `re_render` method on the facet**. Rendering is a private concern of the implementing container, driven off its own content changes: the container consumes its source content's Delta Stream and materializes the projection store incrementally (e.g., Atoms renders affected sections as atoms change). The facet is read-only — it adds no inbound or authoring method.
- **Changing the configured profile requires a full re-projection** — a cold-start replay of the source container's stream from empty (`as_stream()` per [05 — Delta Streams § Full-state stream](../spec/05-delta-streams.md#full-state-stream)), materializing every document afresh under the new profile. This is an operational action a deployer takes, not a runtime API: there is no method that switches profiles in place, and the materialized store under one profile is not partially convertible to another.
- Access control is the perimeter's grant check ([02 — Conformance § Access control](../spec/02-conformance.md#access-control)), identical to every other read surface — the facet methods declare `query` and add no separate authorization model.
- The narrative renderer (where used) calls LLM Gateway with `aala.projection_narrative`. Template-only configurations skip that call and still produce a valid, less fluent document.
