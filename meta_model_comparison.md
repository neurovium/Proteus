# Meta-Model Comparison: Draft vs. Revised (`proteus_meta_model.yaml`)

Notes comparing the revised meta-model against the earlier draft. Context: we need a registry of experimental schemas, an alignment computation over them (automated matching plus human curation), and a knowledge graph that grows as schemas are added.

## What we kept from the draft

- **Content-addressed identity.** Entity IDs are derived from a hash of content, with lineage tracked through `derived_from` instead of version counters. This is the right foundation for a registry and we kept it as-is.
- **Relationships as first-class objects**, not bare pointers.
- **Provenance as its own class.**
- **The graded SKOS mapping vocabulary** (exact/close/broad/narrow/related) instead of a single "same as" relation. The broad/narrow directionality in the draft was also correct.
- **LinkML as the authoring language.** This is what lets non-ontologists contribute.

The revision doesn't change this philosophy; it applies it more consistently and adds what our use case needs.

## Fixes: internal inconsistencies in the draft

Three places where the draft contradicts its own design:

1. **Embedded mappings break content-addressing.** The draft stores `skos_mappings` inside `RegistryEntity`, whose identity is a hash of its content. So adding a mapping changes the entity's content, which changes its hash, which changes its identity — and everything pointing at the old hash dangles. Mappings are also conceptually different from entity content: a mapping is something a third party believes *about* an entity, not part of what the entity *is*. In the revision, mappings are standalone records that reference entities by hash from the outside, so they can be added or revised without touching identity.

2. **`modified_by` / `modified_at` contradict immutability.** The draft's docstring says there's no version slot because a content change produces a new hash — but every entity has modification-tracking slots. Both can't hold. If you modify in place and recompute the hash, that's a new entity, not a modification. If you don't recompute, the ID no longer matches the content. The revision drops the slots: modification means creating a successor entity whose provenance points at the predecessor. This matters for alignment, because a mapping pinned to a hash is only trustworthy if the target can't change underneath it.

3. **Hash scope is undefined, and the natural reading breaks deduplication.** The draft leaves the hash "format TBD," but the real question is what gets hashed. Provenance is required on every entity and includes `created_at`. If provenance is part of the hash, two byte-identical definitions ingested at different times get different identities, and re-registering an unchanged schema mints new entities every time. The revision states explicitly that the hash covers definitional content only, with provenance excluded. (Same rule answers whether an embedded `Relation` is hashed with its subject class: yes, because it's definitional.)

Two smaller fixes:

- `Relation.predicate` was a free string ("isPartOf", "is_part_of", "part of" would be three different predicates). Now a CURIE.
- `derived_from` ranged over free strings rather than identifiers. Now identifiers.

## Additions: what the draft couldn't express

Each of these corresponds to something our objective needs and the draft had no place for.

1. **A home for the alignment computation's output.** The draft's only mapping construct, `SkosMapping`, points outward to external vocabularies — there's no way to assert a mapping between two entities inside the registry. It also carries no confidence score, method, evidence, provenance, or review state, so it couldn't hold a computed mapping even if redirected inward. The revision adds a full mapping record: subject, graded predicate, object, a controlled justification vocabulary (lexical, embedding-based, structural, manual, ensemble), confidence in [0,1], producing tool, matched evidence strings, author/reviewer attribution, and lifecycle status. The alignment pipeline now has a target format.

2. **Registered schemas.** The draft has no class for a schema — entities float free, with nothing recording which schema they came from, at which version, in what format. That blocks three things we need: one named graph per schema version (the unit of loading, export, and access control); diffing on re-registration (so only changed entities re-enter matching); and the hub strategy (the hub is meant to be a registered schema with distinguished status, but there were no schema objects to distinguish). The revision adds `SourceSchema` and `SourceSchemaVersion`, a required `defined_in` on every entity, and a boolean marking the hub.

3. **Structured datatypes and units.** The draft typed properties as `value_type: string` and `units: string`. For alignment, datatype and unit compatibility are strong signals: matching datatypes are cheap corroboration, incompatible ones (float vs. enumerated string) refute a name-based match, convertible units (degrees/radians) support an exact match after conversion, and incommensurable units veto a match regardless of lexical similarity. Free-text units make none of this checkable ("deg", "degrees", "°" are unrelated strings). The revision types `value_type` as a datatype identifier or `ValueSet` reference, and makes units structured objects resolving to standard unit vocabularies.

4. **More text for matchers.** The draft gives each entity one `name`. Lexical and embedding matchers need synonyms, abbreviations, and especially the verbatim source field name (e.g. `z_spec`), which the draft discarded at ingestion. The revision adds multivalued `aliases` and a `source_native_id` preserved exactly.

5. **A curation lifecycle.** Our workflow is machine-proposed, human-promoted: matchers generate candidates, experts accept or reject, queries filter by status and confidence. The draft has no status anywhere — a mapping either exists or doesn't, and rejection isn't representable at all. That's a double loss: rejections are negative evidence for calibrating matchers, and without a record of them the same false match gets re-proposed by every future run. The revision gives every mapping a `review_status` (proposed, accepted, rejected, superseded) and keeps rejections permanently.

6. **SSSOM conformance.** The draft's mapping structure was homegrown. The revision is slot-for-slot conformant with SSSOM, the community standard for this artifact. Registry mappings round-trip through existing tooling (conversion, diffing, inversion, TSV exchange), outputs of off-the-shelf ontology matchers convert directly into registry records, and our mapping sets are publishable for other projects to consume.

7. **Constraints as associations; reusable properties.** The draft parked validation constraints in an unspecified `Rule` stub and inlined properties into classes. But requiredness and cardinality aren't facts about a property in isolation — the same property can be mandatory on one class and optional on another — and inlining means a shared property exists as two unrelated copies, hiding exactly the structural overlap alignment algorithms exploit. The revision replaces most of `Rule` with `SlotAssignment`, an association object carrying class-local constraints, so one property definition is shared by reference and the sharing is visible in the graph.

## Summary

With the revision: the alignment computation has a standards-conformant output format with evidence, confidence, and attribution; the hub strategy is expressible; human curation has a mechanism and rejections accumulate as negative evidence; datatype and unit structure make the strongest matching signals computable; and schema registration plus consistent content-addressing give us the incremental-growth property — when a schema evolves, unchanged elements keep their identities and accepted mappings, and only what changed re-enters the matching queue.
