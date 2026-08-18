# The Alignment Computation: v4 Registry Interface (delta to Pipeline Design v2)

*Companion to `proteus_meta_model_v4.yaml`. The computation itself is unchanged: `alignment_pipeline_design_v2.md` remains the specification for all seven signals, the six stages, composition, and the build order. This document records only what meta-model v4 changes at the pipeline's interface to the registry — where each stage reads from and writes to under the v4 classes and slots, and the two places where a v4 mechanism sharpens a stage's behavior. Read the two documents together; where they overlap, v2 governs the computation and this delta governs the registry contract.*

## The identity contract (affects every stage)

The hash field set is no longer prose. Any code that computes, verifies, or reasons about `hash_id` derives the field set from the meta-model's `in_subset: HashSubset` tags — never from a hardcoded list. The contract that follows from v4's scope decision:

- **Hashes are schema-anchored and version-free.** `defined_in_schema` is hashed; the schema version is not. Therefore hash_ids are unique per (content, schema) and stable across versions of that schema. Two consequences the pipeline relies on: no candidate pair ever consists of one merged cross-schema node (the alignment problem is always well-posed as A-side vs. B-side), and an element unchanged across a re-registration keeps its hash_id, so its accepted mappings remain valid without any migration step.
- **Canonicalization rules** (from the meta-model, design principle 6): multivalued identity slots are passed explicitly and canonically ordered; identity booleans are materialized (`ifabsent` false, never null); entity references inside the hash are by hash_id, so hashes compose Merkle-style.

## Stage 0 — Matching profiles

**The anchor set has exactly one definition now.** In v2's prose, anchors came from "ontology annotations (`exact_mappings`, `close_mappings`, ...)". Under v4 those annotations do not survive ingestion as slots; they are materialized as ordinary Mapping records. The anchor set of entity *e* is:

    anchors(e) = { e.declared_uri }
               ∪ { m.object_id : m.subject_id = e.hash_id,
                                  m.object_id is an external IRI,
                                  m.mapping_justification = MANUAL_MAPPING_CURATION (ingestion-declared)
                                  or m.review_status = ACCEPTED }

`declared_uri` is the element's own declared identity (the source's `class_uri`/`slot_uri`); the mapping-derived members are the source's declared correspondences plus curated outward mappings. PROPOSED statistical mappings never contribute anchors — an anchor is an assertion of intent, and a matcher's own guess feeding back in as "intent" would be circular. Anchor enrichment, the anchor index, tokenization, and the abbreviation dictionary proceed exactly as specified in v2.

Profile assembly reads: `name`, `source_native_id`, `aliases` (all hash-scoped, so guaranteed stable for a given hash_id), `definition`, parent class name via `parent_class`, sibling property names via the class's `slot_assignments`, `value_type`, and the unit (next section).

## Stage 1 — Blocking

**The unit veto gains a cheap first tier.** With v4's merged `UnitOfMeasure`:

1. *Quantity-kind tier:* if both sides carry `has_quantity_kind` and the IRIs name different quantity kinds, veto immediately — no dimension-vector resolution, no conversion lookup.
2. *Dimension-vector tier:* otherwise resolve to QUDT dimension vectors — directly via `qudt_unit` when present, else via `ucum_code` → QUDT resolution — and veto incommensurables as in v2.

Everything else about the veto is unchanged and remains invariant 1: it is a hard filter, never a score; vetoed pairs are logged; a shared anchor plus a veto at either tier is logged at the highest priority.

## Stage 2 — Signals

The *type and unit* categorical features now operate on canonical values: `value_type` is an XSD CURIE or ValueSet hash_id by construction, so there is no type-name normalization at match time — that is ingestion's job, and match-time code must not compensate for ingestion failures by fuzzy-matching type names. The *declared semantics* features read from the Stage 0 anchor set as defined above; the reasoner-materialized index and the entailment features are unchanged from v2.

## Stage 3 — Combination

Unchanged. One restatement for precision: invariant 4 (missing is not zero) applies to `declared_uri` exactly as to any other anchor evidence — most entities will have none, and its absence is a masked feature, never a 0.

## Stage 6 — Write-back and idempotence

v2's idempotence rules are now grounded in concrete registry operations:

- **Re-registration diff is a manifest set-difference.** A new `SourceSchemaVersion` carries `entities`, the manifest of hash_ids extracted from it. The matching queue for a re-registration is `manifest(v_new) − manifest(v_old)`: only genuinely new hash_ids enter. Unchanged elements appear in both manifests under the same hash_id and are never touched.
- **The Merkle cascade is intended behavior.** Editing a property yields a new property hash and, in turn, new hashes for every class whose `slot_assignments` reference it. Those classes legitimately re-enter class-level matching — do not "optimize" the cascade away; the class's structural evidence changed.
- **Successor carry-forward.** When a changed element's predecessor (via `derived_from`) carries ACCEPTED mappings, the run proposes corresponding mappings for the successor with the predecessor mapping referenced in `mapping_comment`, so the reviewer confirms in one action instead of re-adjudicating from scratch. This covers the deliberate v4 case where an alias-only edit mints a successor: lexical evidence changed, so the pair re-enters review, but cheaply.
- **The REJECTED negative cache is keyed by hash pair.** A successor pair is a *new* pair and is not auto-blocked by the predecessor pair's rejection — the content changed, so the rejection may no longer apply — but the predecessor rejection is surfaced to the reviewer alongside the new proposal.
- Mapping subjects and objects are always hash_ids (or external IRIs); `subject_label`/`object_label` are denormalized at write time from the entities' `name`, per SSSOM.

One run still produces one `MappingSet` with tool, version, and parameter hash in the description; everything lands `PROPOSED`; per-justification evaluation is unchanged.

## What this delta does *not* change

Candidate generation channels and the recall target; all seven signal definitions; personalized-PageRank structural diffusion and the Hungarian assignment; calibration (including conditional calibration per evidence regime); predicate-assignment policy and its asymmetries; the Stage 5 coherence checks and the reasoner repair loop (including which predicates translate to OWL); hub composition and the mapping-graph diffusion's review-only role; the concrete stack; the build order. If an implementation question is not answered here, the answer is in `alignment_pipeline_design_v2.md`.
