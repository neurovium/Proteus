# PROTEUS

## Provenance-Rich, Ontology-agnostic Typed Entity Unification System

### A federated knowledge graph for experimental science, built without requiring consensus

### Summary

In any active experimental field, research groups design their own data schemas — organically, independently, and each perfectly serviceable for its own experiment. Some schemas carry links to formal ontologies; most do not, because most scientists are not, and should not need to be, ontologists. The result is a field whose data cannot be queried across experiments: nobody can say with certainty, mechanically, whether a quantity in one group's schema is the same as, related to, or distinct from a quantity in another's. This project builds the infrastructure that answers that question at scale. It is a **schema registry** — a graph database in which schemas are registered exactly as they organically exist — coupled to a **mapping commons**: a growing, evidence-bearing, human-curated layer of computed correspondences between schema elements. 

Together they yield a field-wide knowledge graph that new experiments join incrementally: register a schema, receive machine-proposed mappings, confirm or reject them with domain judgment, and arrive already connected to everything that came before. Throughout this description we use neurophysiology as the running example (e.g. low-pass filtered extracellular voltage appearing as `lfp` in one lab and `local_field_potential` in another, just as in cosmology `z_spec` might appear in one survey and `redshift_spectroscopic` in another), but the framework is deliberately domain-independent: nothing in its design is specific to any one field's semantics.

### The name

In Greek mythology, Proteus is the shape-shifting old man of the sea: he evades every questioner by changing form, and yields the truth only to the one who holds him fast. The image is the project. Experimental/data schemas are protean — the same quantity shifts shape from lab to lab and version to version — and PROTEUS extracts the truth about their correspondences by holding each shape fast: every registered form is pinned by an immutable, content-derived identity, and the truth is won by patient grip — the evidence-weighed, human-curated alignment loop. The Greek *prōtos*, "first," fits too: a registry is the thing everything else anchors to. Each word of the backronym names a design commitment defended in the architecture below: **p**rovenance-**r**ich (nothing is anonymous), **o**ntology-agnostic (no formal-semantics literacy is asked of schema authors), **t**yped (structured datatypes and units as first-class alignment evidence), **e**ntity **u**nification (the mapping commons itself) **s**ystem.

### The problem, stated precisely

The obstacle to a field-wide knowledge graph is not technical storage — it is semantic heterogeneity without ground truth. The same physical quantity appears under different names, different units, and different implicit conventions across schemas; conversely, identically named fields can denote incompatible things. The traditional remedy — convene the community, legislate a unified ontology, mandate adoption — has failed repeatedly and predictably, because it demands consensus before delivering value and asks working scientists to become knowledge engineers. Any viable design must accept three constraints as fixed: schemas will continue to be created organically; their authors will not learn ontology languages; and no moment of field-wide agreement will ever arrive to build upon.

### The central design commitment: agreement is data, not a prerequisite

The project inverts the traditional order. Schemas are never merged, harmonized, or corrected; they are registered faithfully and permanently as they are. What the system then accumulates is *claims about correspondences* between their elements — each claim graded (exact match, close match, broader, narrower, related), scored with a calibrated confidence, tagged with the method that produced it and the evidence it rests on, attributed to its author, and carried through an explicit curation lifecycle from machine proposal to human acceptance or rejection. Semantic agreement is thus an *output* of the system, accumulating claim by claim, rather than an input demanded of the community. The umbrella vocabulary that unifies the field emerges from validated mappings instead of being imposed before them.


### Architecture

The system has five interlocking components, each specified in a dedicated design document.

*   **The registry meta-model.** A formally specified model (authored in LinkML, mechanically compiled to OWL, SHACL, JSON-LD, and typed Python) defining what can be registered and how. Its two load-bearing principles: identity by content — every registered class and property is identified by a hash of its definitional content, so entities are immutable, versioning reduces to lineage links between successive entities, and deduplication is automatic; and a strict wall between the definitional and the epistemic — what a source schema asserts lives inside hashed entities, while what anyone believes about correspondences lives in standalone mapping records that reference entities from outside, so beliefs can accumulate and be revised without ever disturbing identity. Registered properties carry structured datatypes and units (resolvable to standard unit vocabularies), rich textual evidence (definitions, aliases, verbatim source field names), and full provenance.

*   **The mapping layer.** Mappings conform to SSSOM, the community standard for semantic mappings, making every mapping set an exchangeable, tool-compatible artifact rather than a private format. Each mapping is a first-class, citable claim; rejected mappings are retained permanently as negative evidence. A review-status lifecycle (proposed, accepted, rejected, superseded) lets every consumer of the graph choose its own rigor — conservative queries traverse only human-accepted exact matches; exploratory queries admit machine proposals above a confidence threshold.

*   **The hub topology.** Rather than matching every schema against every other (quadratic in effort and degrading through chains of approximation), each schema is matched once against a deliberately minimal core vocabulary — the hub — and schema-to-schema mappings are *derived* by composition under strict rules that refuse to launder uncertainty (two "close" matches do not compose). The hub is not a separate mechanism: it is itself a registered schema with distinguished status, built from the same primitives as everything under it, and grown conservatively — a concept is added only when multiple schemas demonstrably need it. Adding the field's N-th schema costs one matching campaign, not N−1.

*   **The alignment computation.** A multi-stage pipeline proposes mappings automatically. Candidate pairs are generated at high recall by dual lexical and embedding retrieval, then filtered by a dimensional-analysis unit veto — when names descibe physical measurements, no name similarity survives incommensurable physical dimensions. Surviving pairs receive a vector of independent signals (lexical, semantic, type/unit, value-distribution, and structural — whether neighborhoods map to neighborhoods), optionally adjudicated by a large language model as one juror among several. Signals are combined by a model trained on the accumulating curation judgments and calibrated so that a confidence of 0.8 empirically means an 80% chance of correctness — the honesty condition the query semantics silently depend on. A coherence-repair stage resolves global inconsistencies before results land in the registry as proposals. The pipeline and registry form a closed loop: curation decisions are the training labels that continuously improve the matcher.

*   **Human-in-the-loop curation.** Domain experts are asked exactly one kind of question — "is your `lfp` their local field potential: yes, sort of, or no?" (or "is your `z_spec` their spectroscopic redshift?") — presented with the machine's evidence. They never touch ontology tooling. Their accepts and rejects simultaneously build the trusted mapping layer and teach the matcher, so expert attention, the system's scarcest resource, compounds rather than dissipates.

*   **Storage.** The source of truth is version-controlled LinkML; the graph store is a derived, rebuildable index chosen for four capabilities: named graphs (one per schema version, one per mapping set — the partitioning that makes query rigor a graph-selection choice), SPARQL, SHACL validation at the ingest boundary, and text/vector search for the review interface. The recommended configuration is GraphDB Free (or Apache Jena Fuseki as the fully open-source alternative), with an embedded RDF engine in the ingestion pipeline and an optional, disposable property-graph projection for graph algorithms. Reasoning is deliberately restricted: the equivalence closure is materialized explicitly over accepted exact matches only, never inferred globally, because approximate similarity is not transitive.

### Why this design grows instead of decaying

Integration projects typically decay because every change threatens past work. Here, three properties make growth monotone. Content-addressing means a schema update re-mints identities only for elements that actually changed — unchanged elements keep their accepted mappings automatically, and only the delta re-enters the matching queue. The definitional/epistemic separation means new beliefs never perturb old identities. And the open-world, assertion-only semantics of the triple representation mean the graph only ever gains statements; nothing is erased, only superseded, so every past measurement remains reproducible against a stable substrate. The marginal cost of connecting the next experiment *falls* over time, as the hub matures, the abbreviation dictionary and negative-evidence cache grow, and the matcher trains on an ever-larger curated corpus.

### What distinguishes this from prior approaches

*   **Against grand-unified-ontology efforts:** no consensus is required, no adoption is mandated, and values are delivered from the first registered schemas and expanded as new schemas are added.
*   **Against ad-hoc pairwise data integration:** every correspondence is a governed, evidenced, revisable record in a standard format rather than a hard-coded assumption in someone's analysis (or storage or definition) script.
*   **Against pure-ML schema matching:** machine output is never trusted directly — it is calibrated, coherence-checked, and gated through expert review, with the physics of the domain wired in as a hard constraint that statistical methods cannot override. 


The system's epistemics are explicit end to end: it always knows what it believes, how strongly, on what evidence, and on whose authority.

### Deliverables and status

The design phase is complete and documented in six artifacts: the machine-validated registry meta-model (`registry_meta_model.yaml`, compiling cleanly to OWL, SHACL, and JSON-LD) with a pedagogical guide and a comparison against the initial draft that motivated it; the alignment pipeline design with its pedagogical rationale; a storage decision memo; and a primer on the triple representation for engineering new to RDF. 

Implementation follows a deliberately staged build order: first the ingestion pipeline, blocking, lexical and unit signals, and the curation loop — the components that begin accumulating the expert-validated mappings everything else depends on; then embeddings, structural matching, learned combination, and calibration; then hub bootstrapping from existing community vocabularies and the composition machinery. The guiding rule of the roadmap mirrors the guiding rule of the design: build the part that makes agreement accumulate first, and let sophistication be funded by the data that accumulation produces.


### The vision, in one paragraph

A field's collective knowledge should be a queryable object, not a collection of literature. This project makes it one without asking the field to change how it works: experimentalists keep designing the schemas their experiments need, and the registry does the connecting — transparently, with every inference graded, evidenced, and reviewable. Each new experiment that registers makes every past experiment more valuable, and the shared understanding of the field crystallizes where it should: not in a committee's document, but in an open, growing, machine-readable commons of validated correspondences.
