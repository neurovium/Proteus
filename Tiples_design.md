### Triples: Atoms of assertion

**The atom of assertion**
A triple is the smallest complete statement a machine can hold: a subject, a predicate, and an object, in that order. Read it as a minimal sentence.

```text
:lfp  :hasUnit  :Microvolt .
```
"The thing/entity called `lfp` has-unit microvolt." (Or for cosmology: `:z_spec :hasUnit :Dimensionless .`) This clarifies `one fact`, no more. Something richer that one may have defined in a schema — "`lfp` is a required float-valued property of the `ExtracellularRecording` class, defined as the low-pass filtered voltage" (or "`z_spec` is a required float-valued property of the `SourceCatalogEntry` class, defined as the spectroscopic redshift") — is not one triple but several:

```text
:lfp         rdf:type        proteus:RegistryProperty ;
             skos:prefLabel  "lfp" ;
             skos:definition "Low-pass filtered extracellular voltage" ;
             :valueType      xsd:float .
:assign_17   :property       :lfp ;
             :required       true .
:ExtracellularRecording :slotAssignment :assign_17 .
```

This decomposition — every fact broken down to atomic subject–predicate–object statements — constitutes the basis of the model. A knowledge graph, in the RDF sense, is bascically a set of such triples. There is no additional machinery hiding behind the curtain: no tables, no objects with fields, no documents. Everything the registry knows, from the definition of `lfp` (or `z_spec`) to the confidence score on a mapping to the provenance of a curator's judgment, is expressed in this one grammatical form. 

**Names that work across**
Note that the subjects and predicates above are not strings; they are identifiers — IRIs, abbreviated as CURIEs (`skos:prefLabel` expands to a full web address). Objects can be identifiers too, or they can be literals: strings, numbers, dates, the leaves of the graph where actual values live.

The insistence on globally scoped identifiers, rather than local names, is the first place the triple model earns its keep for us. In a relational database, `lfp` (or `z_spec`) is a column name meaningful only inside one table inside one schema inside one lab's system; the moment two labs' databases meet, their names collide or talk past each other. In RDF, Team A's `voltage` property and Team B's `voltage` property (or Team A's and Team B's `redshift` properties) are different identifiers by construction — each minted in its own namespace (in our registry, each is a content-hash CURIE anchored to its source schema version) — and any claim that they denote the same quantity must be made explicitly, as data. Nothing is ever accidentally conflated by a name clash, and nothing is ever conflated implicitly at all. For our project whose entire subject matter is "when are two independently named things the same?", a representation in which identity-of-name never silently implies identity-of-thing is not a convenience. It is the precondition of the enterprise.

**Merging is union; growth is monotone**
Because a graph is just a set of triples with globally scoped names, combining two graphs is set union. When we load Team A's transcribed schema; load Team B's; nothing needs to be reconciled, migrated, or renamed for the two to coexist in one store. Compare the relational alternative: merging two databases means designing a shared schema first — which is precisely the consensus our field does not have and this project refuses to require. The triple model lets us postpone semantic agreement while still physically integrating the data, which is the registry's founding move.

The same property gives us monotone growth. Adding triples never invalidates existing triples; there is no schema to migrate, no table to alter, no document to rewrite. A new experiment registers, a matcher run lands, a curator rules — each event only ever `adds` statements. The knowledge graph, that grows as the field does and as we ingest more schemas, is in the triple model which is the default behavior of the representation.

The schema is made of the same stuff as the data. The statement "`lfp` is a RegistryProperty" and the statement "RegistryProperty is a class in the registry meta-model" are both just triples, in the same store, queryable by the same language. This self-description is why a registry of schemas is natural in RDF — our data literally is other people's schemas, and the triple model doesn't distinguish levels: describing a schema, an action potential, or a galaxy are the same grammatical act.

**The notion of `statements about statements` in our system**
Our central object is not a fact but a claim about a correspondence: "A's `lfp` closely matches B's `local_field_potential`" (or "A's `z_spec` closely matches B's `redshift_spectroscopic`") "— with confidence 0.91, proposed by an embedding matcher on this evidence, not yet reviewed." Notice the shape: this is not an edge, it is an edge carrying a payload — a score, a method, a lifecycle status, an author. The naive rendering as one triple (`:lfp skos:closeMatch :local_field_potential`) has nowhere to hang the payload; a bare triple is *asserted* or *absent*, with no room for "asserted at 0.91 by a machine, pending review."

RDF's answer, and the pattern our meta-model uses, is to promote the claim itself to a "thing/entity" — a node with its own identifier — and decompose the payload into ordinary triples about that node:

```text
:map_0042  rdf:type                   proteus:Mapping ;
           sssom:subject_id           :lfp ;
           sssom:predicate_id         skos:closeMatch ;
           sssom:object_id            :local_field_potential ;
           sssom:confidence           0.91 ;
           sssom:mapping_justification semapv:SemanticSimilarityThresholdMatching ;
           proteus:review_status      proteus:PROPOSED .
```

The mapping is now first-class: it can be scored, evidenced, attributed, reviewed, superseded, cited — and, critically, quantified over. "How many accepted exact matches anchor schema A to the hub?" "What is the empirical precision of embedding-justified mappings above 0.8?" "Which mappings does this rejected one contradict?" These are ordinary queries over ordinary triples, because the epistemology was reified into the same substrate as everything else. In a representation where relationships are primitive edges rather than describable objects, every one of those questions requires stepping outside the data model. In this one, the registry's beliefs about itself are just more graph.

*(Two footnotes: i. RDF also offers older reification vocabulary and newer RDF-star syntax for annotating edges; we use the explicit-node pattern instead because our mappings carry rich, standardized, multi-field payloads — SSSOM records — and because a mapping's lifecycle demands it be addressable on its own. Notice the connection to the meta-model's definitional/epistemic wall: triples inside an entity's description are definitional and hashed; ii. the mapping node's triples are epistemic and live outside all entities. The triple model doesn't enforce that wall — our design does — but it makes the wall expressible.)*

**Named graphs**
In practice every triple in our databse carries one more coordinate: the named graph it belongs to, making it technically a "*quad*". A named graph is a labeled bag of triples — and we use the labels for provenance-shaped partitioning: one graph per registered schema version, one per mapping set, one for the meta-model, one for the materialized equivalence closure.

This is where quantification meets query mechanics. "Traverse only human-accepted exact matches" is not a special mode of the database; it is a query that selects certain graphs and filters on `review_status` and `predicate_id`. "Drop that miscalibrated matcher run" is the deletion of one named graph. "Rebuild the equivalence closure" drops and reconstructs one graph without touching evidence. Every consumer of the registry chooses its own rigor, because rigor is expressible as graph selection — a direct consequence of triples being individually addressable, individually attributable atoms rather than rows locked inside tables.

**Open world: absence is not denial**
RDF adopts the open-world assumption: a statement not present in the graph is "unknown, not false". A relational database implicitly claims completeness — no row means no fact. A federation of independently curated schemas can claim no such thing; the registry not containing a mapping between two properties means only that nobody has asserted one yet.

This default is a delibrate choice for federated science, and it explains a design decision that otherwise looks odd: why the meta-model insists on keeping rejected mappings as explicit REJECTED records. In an open world, one cannot encode "an expert examined this pair and ruled it out" by deleting the mapping — absence already means merely "unexamined," and precious negative knowledge would be indistinguishable from ignorance. Negative evidence must be asserted, not implied. The open-world stance forces our epistemics to be honest and explicit, which is exactly the discipline a machine-and-human belief system needs.

**Design choice for what we want to quantify**
Our project's quantitative targets are: similarity signals between schema elements; calibrated confidences on correspondence claims; structural evidence (do neighborhoods map to neighborhoods?); compositions of mappings through a hub with controlled uncertainty; coverage and agreement statistics over the whole registry; and the evolution of all of the above as schemas change and curation accumulates. Each target leans on a specific property of the triple style.

**Structural signals need uniform, atomic edges.** The pipeline's structural matcher asks whether the neighborhood of one entity maps onto the neighborhood of another — a computation over graph adjacency. Because every fact is an edge of the same atomic kind, "neighborhood" is well-defined and cheaply extractable for any entity, whether it came from a FITS convention or a Neuropixels NWB extension. Representations that bundle facts into records make the graph structure implicit; triples make it the native geometry.

**Confidence must attach to claims, so claims must be things.** Calibrated probability is our contract with every downstream query, and probability needs a bearer. The reified-mapping pattern gives every claim an identity to which confidence, justification, evidence, and status attach as data — quantification about the graph stored in the graph, queryable alongside it.

**Composition is path algebra over typed, inspectable hops.** Deriving A↔B from A→hub→B is a walk over mapping records whose predicates and confidences are readable at every hop — which is what lets us enforce the composition table (exact∘exact = exact; close∘close = do not derive) and refuse to launder uncertainty. In a representation where links are opaque or untyped, composition either can't see the grades or can't be stopped from chaining them.

**Registry-level statistics are counting queries.** Hub coverage per schema, precision per justification type, disagreement rates between composed and direct mappings, curation throughput — all reduce to counting triples matching patterns, because every event in the system's life deposited countable, attributable atoms. The registry's health metrics come for free from its representation.

Note that **monotone growth** underwrites all of it longitudinally. Because assertions accumulate without invalidating each other, and because content-addressed identities never shift, every quantity above can be tracked over time against a stable substrate. The graph in five years contains today's graph; today's measurements remain reproducible against it.

A closing contrast makes the choice concrete. A relational store would demand the up-front unified schema whose impossibility motivated the project. A property graph (like Neo4j-style) is genuinely graph-shaped and easy to compute over — which is why we use one as a derived projection for algorithms — but its edge properties are local conveniences, not globally identified, standards-bound, self-describing assertions; it has no native notion of federated identifiers, no SHACL contract, no open-world semantics, and its mappings-with-payloads would be a private idiom rather than exchangeable SSSOM records. The triple style is the one representation in which the correspondence claims themselves — graded, evidenced, attributed, revisable — are ordinary citizens of the same graph they describe. Since quantifying those claims is the project, the project speaks in triples.

Summary: the registry is a lab notebook that machines can read — every entry an atomic, signed, dated assertion; every conclusion traceable to entries; nothing erased, only superseded. Triples are simply what such a notebook's lines look like. 
