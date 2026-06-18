# Appendix B — OKF Profile

This appendix is **normative**. It defines `okf`, the default [projection output profile](./appendix-a-registries.md#projection-output-profiles) — the rendered form the Projection facet's materialized store takes when a deployment does not override it. The profile maps aala state onto the **Open Knowledge Format (OKF)**, a permissive directory-of-markdown interchange format for knowledge bundles.

The profile contract every output profile satisfies is enumerated in [Appendix A § Projection output profiles](./appendix-a-registries.md#projection-output-profiles); this appendix is `okf`'s satisfaction of it. The single-profile-per-deployment rule, the determinism tuple, and the re-projection cost of changing the configured profile are owned by the [Projection facet contract](../interfaces/projection.md) — this appendix defines only the OKF rendering, not when or whether it is used.

## Versioning

This profile is **versioned independently** of the aala interface and spec versions. `aala-v0.1`'s default profile targets **OKF 0.1**. The profile version tracks the OKF version it targets: a new OKF release moves this profile to that version, and the profile-version change is *not* an aala interface-version change ([Projection § Version notes](../interfaces/projection.md#version-notes)). This insulation is deliberate — OKF evolving does not ripple a breaking change through the aala contract; only [`ProjectionDoc.profile`](../interfaces/projection.md#read) (still `okf`) and the rendered document format move.

OKF's own compatibility rule is inherited: an OKF minor bump is backward-compatible, a major bump is breaking. The bundle declares the targeted OKF version in its root index (see [§ Bundle](#bundle)).

## What OKF is

An OKF knowledge bundle is a directory tree of UTF-8 markdown files. Two filenames are reserved — `index.md` (a directory listing) and `log.md` (a chronological update history) — and every other `.md` file is a **concept document**. A concept's identity is its file path with the trailing `.md` removed (e.g. `tables/users.md` names the concept `tables/users`). Each concept document is YAML frontmatter plus a markdown body; OKF requires only the `type` key and recommends `title`, `description`, `resource`, `tags`, and `timestamp`. Links between documents are bundle-relative paths; a link asserts *a* relationship, but OKF does not type it — the kind is conveyed by surrounding prose. OKF is deliberately permissive: producers may add any keys, consumers must tolerate unknown ones and broken links, and it is an authoring/interchange format, not a structured claims database.

aala renders *into* OKF. The structured typing OKF leaves to prose, aala holds authoritatively in its Atoms store; the OKF bundle is the readable, portable **shadow** of that store. Where OKF is loose, aala populates the recommended fields fully and from real structure (see [§ Lossy downcast](#lossy-downcast)).

## Bundle

The OKF bundle **is** the materialized projection store: a directory tree of markdown documents the facet keeps in sync with its source content's Delta Stream ([Projection § Output profile](../interfaces/projection.md#output-profile)). The bundle root carries the two reserved files:

- **`index.md`** ← the [`ProjectionIndex`](../interfaces/projection.md#read_index). The hierarchical index — node titles, one-line summaries, and child structure — serializes as the OKF directory listing: each node renders as an entry with its `title`, its `summary` as the description, and its `children` as nested listings. Subdirectory `index.md` files render the index subtree rooted at that directory. The bundle root's `index.md` frontmatter declares `okf_version: "0.1"`.
- **`log.md`** ← the facet's Delta Stream. The chronological history serializes the projection events ([`changes_since`](../interfaces/projection.md#changes_since) / [`as_stream`](../interfaces/projection.md#changes_since)) — `DocAdded`, `DocChanged`, `DocRemoved`, `IndexUpdated` — newest-anchored, each entry naming the affected document path and its `timestamp`. A cold-start re-projection replays `as_stream()` ([05 — Delta Streams § Full-state stream](./05-delta-streams.md#full-state-stream)); the log reflects that materialization order.

## Concept documents

A concept document ← a [`ProjectionDoc`](../interfaces/projection.md#read). The concept's identity (path minus `.md`) ← the document's [`SectionPath`](../interfaces/00-shared-types.md#path-shaped-strings-different-shape): `domains/payments/checkout.md` is the concept `domains/payments/checkout`, addressed in aala as the `SectionPath` `domains/payments/checkout.md`. `ProjectionDoc.content` is the full rendered document (frontmatter + body); `ProjectionDoc.frontmatter` is the parsed structured head below.

A concept document **MAY** render from many atoms — Atoms projects a coherent section (a service, a decision, a domain concept) that several atoms jointly describe. The document-level frontmatter therefore names a **primary subject**: the atom the section is principally about. Document-level scalar fields (`type`, `title`, `description`, `timestamp`, `resource`) derive from that primary subject; the other rendered atoms contribute body content and citations.

### Frontmatter mapping

The frontmatter is deterministic and machine-parseable ([Projection § Determinism](../interfaces/projection.md#determinism)); a caller reads it via [`ProjectionDoc.frontmatter`](../interfaces/projection.md#read) without parsing the body. OKF's recommended keys map as:

| OKF key | aala source |
|---|---|
| `type` (OKF-required) | the primary subject's `ClassificationAtom` display name — the classification's human-facing name, from its place in the registered classification hierarchy ([03 — Data Model § Standard library](./03-data-model.md#standard-library--pre-registered-atoms-per-tree)). |
| `title` | the primary subject atom's label (its `subject`/identity rendered for display). |
| `description` | the primary subject's one-line summary — the same gist the [`ProjectionIndex`](../interfaces/projection.md#read_index) `summary` carries for the node. |
| `resource` | the primary subject's provenance source URI when present — a `Source.uri` ([03 — Data Model § Provenance](./03-data-model.md#provenance--source)) of the underlying asset; omitted when no asset URI applies. |
| `tags` | the primary subject's `annotations` rendered as a tag list (free-form record metadata; never reasoning input — [03 § Base Atom fields](./03-data-model.md#base-atom-fields)). |
| `timestamp` | the primary subject's `status.changed_at`, an [RFC 3339](https://www.rfc-editor.org/rfc/rfc3339) `date-time` ([00 — Shared Types § Timestamps](../interfaces/00-shared-types.md#timestamps-and-durations)). |

aala populates `type` **authoritatively** from its real classification hierarchy. OKF treats `type` as a free, centrally-unregistered string consumers must tolerate as unknown; aala's value is richer — it is the display name of a registered ClassificationAtom, with the full `is_a` lineage available through the Atoms API behind the document.

aala-specific structure that has no OKF recommended key is carried under an **`aala.`** namespace (OKF permits arbitrary producer keys, and consumers preserve unknown ones). The namespace carries the rendered atom ids and the classification tree of the document's subject — e.g. `aala.atom_ids` (the [`provenance`](../interfaces/projection.md#read) list), `aala.primary_subject` (the primary subject's `AtomId`), `aala.classification` (the subject's `is_a` chain). These are the typed handles an agent uses to cross from the readable bundle back into the structured store via [`Atoms.get_by_ids`](../interfaces/atoms.md#get_by_ids).

### Body

The body favors structural markdown under OKF's conventional headings:

- **`# Schema`** — the classification and attribute structure where relevant: the subject's classification, its declared attribute slots and their values, and required relations, rendered as readable structure rather than raw types.
- **`# Examples`** — illustrative renderings where the source content carries them.
- **`# Citations`** ← the rendered atoms' [`provenance`](../interfaces/projection.md#read). The atoms the document rendered from appear as **bundle-relative links** to their own concept documents (so a reader navigates from prose to the atom's own page); the original fragment sources behind those atoms (`Source.uri` / `Source.quote` — [03 § Provenance](./03-data-model.md#provenance--source)) appear as numbered **external citations** at the bottom, OKF-style.

### Links

Inter-document links are **bundle-relative** (rooted at the bundle, OKF's recommended form). A `RelationAtom` renders as a link from the subject's document to the object's document — the relation asserts a relationship, which is exactly what an OKF link asserts.

#### Lossy downcast

OKF links are **untyped**: a link records that A relates to B, and the *kind* of relationship is conveyed only by the surrounding prose. aala edges are **typed** — every RelationAtom resolves to one of the ten predicate kinds ([Appendix A § Predicate kinds](./appendix-a-registries.md#predicate-kinds)), with hard characteristics and cascade behavior. Rendering a RelationAtom as an OKF link therefore **drops the predicate kind**: the projection writes the kind into the surrounding prose (and the `aala.` frontmatter retains the source atom ids), but the link itself carries no type.

This is a deliberate **downcast**. The OKF bundle is the readable shadow; the **structured Atoms API retains the full typing** — an agent that needs the predicate kind reads it from [`Atoms.traverse_relations`](../interfaces/atoms.md#traverse_relations) behind the document, not from the link. No information is *lost from aala*; it is merely not expressible in the OKF link.

Broken links are tolerated, aligning with both formats: OKF requires consumers to tolerate links to not-yet-written knowledge, and aala renders a link to an atom whose concept document is absent (e.g. a reference to an atom outside the projected scope, or a dead AtomId reference) as a link the reader may find unresolved — consistent with aala's dead-reference handling ([12 — Edge Cases § Dead AtomId references](./12-edge-cases.md#dead-atomid-references)).

## Determinism and conformance

Rendering under this profile is deterministic on the tuple `(content_ref, profile, template_version, model)` ([Projection § Determinism](../interfaces/projection.md#determinism)): with `profile = okf` fixed, an unchanged source content ref renders byte-identical frontmatter and body. The frontmatter ordering and serialization are stable so that an unchanged document produces an unchanged file and real diffs stand out.

A deployment running the default profile satisfies the Projection conformance obligation ([02 — Conformance § Required facet — Projection](./02-conformance.md#required-facet--projection)) by materializing its store as an OKF 0.1 bundle per this appendix and advertising `projection_profile: "okf"` in its capability declaration ([`OrchestrationCapabilities`](../interfaces/orchestration.md#capabilities)).
