# kg-pipeline

Knowledge graph construction for the ELITE Portal, contributing ELITE's public portal metadata (Layer 1) to SageBrain. Modeled on [nf-osi/kg-pipeline](https://github.com/nf-osi/kg-pipeline) and [amp-als/kg-pipeline](https://github.com/amp-als/kg-pipeline).

## Contents

| Path | What it is |
|---|---|
| `schema/ontology.ttl` | ELITE ontology (v0.1): classes and properties in the `elite:` namespace, aligned to BioLink |
| `docs/identifier-and-mapping-rules.md` | How portal tables become RDF: IRI rules, field mappings, literal-vs-link rule, scope and data-quality notes |
| `mappings/sssom/` | SSSOM lookups from portal labels to ontology IRIs (species so far) |

## Scope (v0.1)

Studies, files, datasets, projects, programs and publications from the ELITE Portal backend tables. Grant, person and tool entities are deferred; participant-level data is out of scope. See the rules doc for details.

## Related

- EC-855: ELITE knowledge graph for SageBrain
- ADM-8435: scoping and competency questions
- ADM-8620: ontology and identifier/mapping rules (this deliverable)
- ADM-8587: ELITE data model LinkML conversion (shares the SSSOM term mappings)