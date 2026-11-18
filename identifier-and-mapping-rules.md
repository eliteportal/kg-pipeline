# ELITE Knowledge Graph: Identifier and Field-Mapping Rules

Version 0.1 · ADM-8438 · Companion to `schema/ontology.ttl`

These rules say how public ELITE Portal metadata becomes RDF: which IRI each entity gets, which columns become which properties, and which values stay literals versus links to ontology concepts. They follow the conventions already used by [nf-osi/kg-pipeline](https://github.com/nf-osi/kg-pipeline) and [amp-als/kg-pipeline](https://github.com/amp-als/kg-pipeline) so the ELITE graph joins cleanly with other portals in SageBrain.

## 1. Scope

Layer 1 only: public portal metadata from the Phase 1 sources agreed in ADM-8435.

| Source | Synapse ID | Table type | Produces |
|---|---|---|---|
| Backend - All Entities | syn51489960 | EntityView | `elite:Study`, `elite:File` (filtered by entity type; see §6) |
| Portal - Projects | syn51209786 | TableEntity | `elite:Project` |
| Portal - Programs | syn73580515 | TableEntity | `elite:Program` |
| Portal - Publications | syn51407023 (current; syn51209321 is not used) | EntityView | `elite:Publication` |
| Portal - Datasets | syn75102003 | EntityView | `elite:Dataset` |

Participant-level records (Cohort Builder tables) and values extracted from data files are out of scope.

All Entities is a mixed view: it holds files, folders, tables, views, datasets and RecordSets. Only study folders and files are extracted from it; datasets come from Portal - Datasets, and the other entity types are excluded (§8).

## 2. Namespaces

| Prefix | IRI | Used for |
|---|---|---|
| `elite:` | `https://eliteportal.synapse.org/terms#` | ELITE classes, properties and minted entity IRIs |
| `biolink:` | `https://w3id.org/biolink/vocab/` | Parent classes for cross-portal alignment |
| `NCBITaxon:` | `http://purl.obolibrary.org/obo/NCBITaxon_` | Species |
| `EDAM:` | `http://edamontology.org/` | Data types |
| `UBERON:` / `NCIT:` / `CL:` | `http://purl.obolibrary.org/obo/…` | Specimen types |

The `elite:` namespace follows the ALS pattern (`https://alskp.synapse.org/terms#`). Decided 2026-11-17: ELITE uses its own `elite:` namespace rather than a shared SageBrain namespace.

## 3. BioLink alignment

Same rule as NF: if a portal table backs the entity, define an `elite:` class, make it `rdfs:subClassOf` the BioLink parent where one exists, and emit **both** `rdf:type` values on every instance.

| Class | BioLink parent | Emitted types |
|---|---|---|
| `elite:Study` | `biolink:Study` | `elite:Study`, `biolink:Study` |
| `elite:Dataset` | `biolink:Dataset` | `elite:Dataset`, `biolink:Dataset` |
| `elite:Publication` | `biolink:Publication` | `elite:Publication`, `biolink:Publication` |
| `elite:File` | none | `elite:File` |
| `elite:Project` | none | `elite:Project` |
| `elite:Program` | none | `elite:Program` |

## 4. Identifier (IRI) rules

**Rule 1: Synapse-backed entities use their Synapse IRI.** Anything that is a Synapse entity keeps the IRI `https://www.synapse.org/Synapse:{id}`, where `{id}` is the `syn` ID. This matches NF and ALS, so the same file or study is the same node in every graph.

**Rule 2: Entities keyed by a name get a minted `elite:` IRI.** Use `elite:{class}/{key}` with the key normalized as below.

**Rule 3: Publications use their Synapse IRI too.** `publicationIds` on studies and datasets is an `ENTITYID_LIST`, so studies already point at publications by Synapse ID; keying publications the same way means those links resolve with no lookup. The PubMed record is linked separately via `elite:pmid`. (Superseded draft: keying by PMID, which only mattered if two Publications tables both fed the graph.)

| Class | IRI template | Key column |
|---|---|---|
| Study | `https://www.synapse.org/Synapse:{id}` | All Entities `id` |
| File | `https://www.synapse.org/Synapse:{id}` | All Entities `id` |
| Dataset | `https://www.synapse.org/Synapse:{id}` | Portal - Datasets `id` |
| Project | `https://eliteportal.synapse.org/terms#project/{shortName}` | Portal - Projects `shortName` |
| Program | `https://eliteportal.synapse.org/terms#program/{program}` | Portal - Programs `program` |
| Publication | `https://www.synapse.org/Synapse:{id}` | Portal - Publications `id` |

**Key normalization for minted IRIs:**

- Trim leading and trailing whitespace.
- Replace internal whitespace with `_` (`NECS APOE` → `NECS_APOE`).
- Percent-encode any other character not allowed in an IRI path.
- Preserve case, so keys stay recognizable and don't collide.
- Don't mint an IRI for placeholder values such as `Not assigned`, `NA` or empty cells; emit no triple instead.
- `CDCP` in a `project` column is treated as a placeholder too: CDCP is a program, not a project, and has no Projects row. Those records should be reannotated with an NA-style project value; until then they get no project link (their program link to CDCP is unaffected).

**Keys that point at another entity resolve to that entity's IRI at extract time.** For example, a file's `studyKey` is looked up against study rows in All Entities to find the study's Synapse ID, and `elite:parentStudy` points at `https://www.synapse.org/Synapse:{that id}`. If the lookup fails, log it as a broken link and emit nothing; never mint an IRI from an unresolved key.

## 5. Literal vs. link rule

A column's values **link to an ontology concept** when both are true:

1. The values are a controlled vocabulary in the ELITE data model, and
2. A Phase 1 competency question filters or groups on them.

Otherwise they stay **literals**. Free-text fields stay literals even when a question uses them, until they've been cleaned.

| Column | Decision | Target | Why |
|---|---|---|---|
| `species` / `speciesName` | Link | NCBITaxon | Controlled; used by questions 1, 3, 7 |
| `dataTypeAll` / `dataType` | Link | EDAM (fallback `elite:`) | Controlled; used by questions 1, 7 |
| `specimenType` | Link | UBERON / NCIT / CL | Controlled; used by question 3 |
| `dataRestriction` | Literal | — | Three-value access flag; candidate for DUO alignment later |
| `dataStatus` | Literal | — | Portal status flag, not a domain concept |
| Institution columns | Literal | — | Free text with no valid values; harmonize to ROR once cleaned |
| `assay`, `fileFormat` | Literal | — | Not used by Phase 1 questions; candidates for later |

Label-to-IRI lookups live in SSSOM files under `mappings/sssom/`, one per vocabulary, matching NF's format. Species is seeded in `species_lookup.sssom.tsv`. The data type and specimen type lookups come from the LinkML epic's term mapping work (ADM-8594 and ADM-8595), which should write SSSOM so the pipeline can read it directly.

A label with no mapping row is logged and emitted as an `elite:` term (`elite:{normalized label}`), never dropped silently.

## 6. Field mappings

`→ IRI` means the value becomes a link; `→ literal` means a typed string, boolean or date. All columns below were checked against the live tables on 2026-11-16.

### Study (Backend - All Entities, `type = folder`, `metadataType = study` and `isReleased = true`)

The `type = folder` filter stops a stray `metadataType` annotation on another entity type from creating a study. Live counts: 25 released and 4 unreleased study folders.

| Column | Property | Value |
|---|---|---|
| `id` | subject IRI | Synapse IRI |
| `studyKey` | `elite:studyKey` | literal |
| `studyName` | `elite:studyName`, `rdfs:label` | literal |
| `studyAbstract` | `elite:studyAbstract` | literal |
| `studyDescription` | not emitted | Holds a Synapse ID on study rows, not a description (see §7) |
| `studyType` | `elite:studyType` | literal, case-normalized (values include `Individual` and `individual`) |
| `project` | `elite:project` | → Project IRI via Portal - Projects `shortName`; `CDCP` and `Not assigned` produce no link |
| `program` | `elite:program` | → Program IRI |
| `relatedStudies` | `elite:relatedStudies` | → Study IRI, multi-value; values are Synapse IDs |
| `publicationIds` | `elite:publicationIds` | → Publication IRI, multi-value; values are Synapse IDs |
| `species` | `elite:species` | → NCBITaxon IRI, multi-value |
| `dataTypeAll` | `elite:dataType` | → ontology IRI, multi-value |
| `specimenType` | `elite:specimenType` | → ontology IRI, multi-value |
| `dataStatus` | `elite:dataStatus` | literal; live values `Available`, `Data Pending` |
| `dataRestriction` | `elite:dataRestriction` | literal, multi-value; values seen include `Registered`, `Controlled`, `Not Applicable` |
| `dataGenerationInstitution` | `elite:dataGenerationInstitution` | literal, multi-value |
| `isModelSystem` | `elite:isModelSystem` | boolean |
| `isReleased`, `metadataType`, `type` | not emitted | used as filters only |
| `grant` | not emitted | Grant deferred (see §8) |

### File (Backend - All Entities, `type = file`, in released studies)

Filter on Synapse's built-in `type` column so folders, tables, datasets and RecordSets are never typed as `elite:File`. Live counts: 65,507 files, 1,299 folders, 135 RecordSets, 89 datasets, 21 tables. File metadata is in scope for every released study, including controlled-access studies (decided 2026-11-17): Layer 1 covers the metadata, not the data. Keep only files whose `studyKey` matches a study row with `isReleased = true`. In the 2026-11-16 check, 52,680 of 65,507 files resolved; 5,957 have no `studyKey` and 6,870 have a key matching no study (see §7). Unresolved files are logged and left out. `parentId` is the containing folder, not the study, so it isn't used for the study link.

| Column | Property | Value |
|---|---|---|
| `id` | subject IRI | Synapse IRI |
| `name` | `elite:fileName`, `rdfs:label` | literal |
| `studyKey` | `elite:parentStudy` | → Study IRI via lookup against study rows' `studyKey` |
| `species` (list) or `speciesName` | `elite:species` | → NCBITaxon IRI; use `species` when populated, else `speciesName` |
| `dataType` | `elite:dataType` | → ontology IRI |
| `assay` | `elite:assay` | literal, multi-value |
| `fileFormat` | `elite:fileFormat` | literal |
| `isModelSystem` | `elite:isModelSystem` | boolean |

### Dataset (Portal - Datasets)

Datasets come from Portal - Datasets only, never from All Entities. Dataset entities appear in both, and because both key on the Synapse ID they would land on the same node; a single source avoids partial or conflicting properties.

| Column | Property | Value |
|---|---|---|
| `id` | subject IRI | Synapse IRI |
| `name` | `elite:datasetName`, `rdfs:label` | literal |
| `studyId`, else `studyKey` | `elite:parentStudy` | → Study IRI; use `studyId` (an entity ID) when populated, otherwise resolve `studyKey` |
| `project` | `elite:project` | → Project IRI, multi-value; `CDCP` and `Not assigned` produce no link |
| `species` | `elite:species` | → NCBITaxon IRI (single value here, unlike studies) |
| `dataType` | `elite:dataType` | → ontology IRI |
| `specimenType` | `elite:specimenType` | → ontology IRI |
| `dataRestriction` | `elite:dataRestriction` | literal (single value here) |
| `publicationIds` | `elite:publicationIds` | → Publication IRI, multi-value |
| `assay` | `elite:assay` | literal, multi-value |
| `datePublished` | `elite:datePublished` | date, parsed from string |
| `license` | `elite:license` | literal, multi-value |

### Project (Portal - Projects)

| Column | Property | Value |
|---|---|---|
| `shortName` | subject IRI, `elite:projectShortName` | minted IRI; literal |
| `name` | `elite:projectFullName`, `rdfs:label` | literal |
| `program` | `elite:program` | → Program IRI |
| `projectId` | not emitted | Not a reliable key: one row is `synTBD`, and some study rows' `projectId` points at a different project than their `project` value |

### Program (Portal - Programs)

| Column | Property | Value |
|---|---|---|
| `program` | subject IRI, `elite:programName` | minted IRI; literal (values `ELITE`, `CDCP`, matching all program values in use) |
| `fullName` | `rdfs:label` | literal |

### Publication (Portal - Publications)

| Column | Property | Value |
|---|---|---|
| `id` | subject IRI | Synapse IRI |
| `PMID` (integer; `PubmedId` string as fallback) | `elite:pmid` | → PubMed IRI; `PMID` is often empty, so the fallback matters |
| `DOI` | `elite:doi` | literal |
| `Title` | `elite:publicationTitle`, `rdfs:label` | literal |
| `Year` (string) | `elite:publicationYear` | gYear, parsed |
| `Grant` / `grant` | not emitted | Grant deferred (see §8) |

The table is syn51407023 (653 rows); all `publicationIds` referenced by studies and datasets resolve to rows in it. `Year` is a reserved word in Synapse SQL, so quote it (`"Year"`) in extract queries.

## 7. Multi-value and data-quality handling

- **List columns** produce one triple per value. Split on the table's delimiter, then trim each value.
- **Concept crosswalk:** the data model names some concepts several ways. Each maps to one graph property: `project` / `projectShortName` → `elite:project`; `species` / `speciesName` → `elite:species`; `dataTypeAll` / `dataType` → `elite:dataType`.
- **Duplicates:** every entity is keyed by Synapse ID (or a normalized key for projects and programs), so it can't duplicate within one source. Only one Publications table should feed the graph.
- **Known source data issues (fix at the source, log during build):**
  - `studyDescription` on study rows holds the study's own Synapse ID; the real description is in `studyAbstract`.
  - The NECS study's `studyAbstract` is a Synapse ID, not an abstract.
  - `studyType` mixes `Individual` and `individual`.
  - The data model's `dataStatus` valid values (`Available`, `Not Available`) don't match the live values (`Available`, `Data Pending`). Worth fixing in the LinkML cleanup ticket (ADM-8592).
  - 6,869 files carry `studyKey = Mouse_M005_Study_metabolomics` (lowercase m), which matches no study row; the dataset uses `Mouse_M005_Study_Metabolomics`. One file carries `LC_metabolomics`. Fix the keys at the source rather than matching case-insensitively, which could hide real mismatches.
  - `studyKey` is `MEDIUMTEXT` in All Entities, so it can't be filtered or grouped in Synapse SQL; resolve it in pandas at extract time.
- **Broken links and unmapped labels** are written to a build log so they can be fixed at the source, not patched in the graph.

## 8. Deferred and excluded

| Item | Status | Reason | Revisit |
|---|---|---|---|
| Grant | Deferred | Decided for v0.1. Grant strings are inconsistent across tables (`grant`, `grantNumber`, `grants`) and need cleanup before they can be a reliable entity. | v0.2, after grant normalization |
| Person (Portal - People) | Deferred | Phase 3 per ADM-8435; needs a decision on people in a shared graph | Phase 3 |
| Computational tool | Deferred | Phase 2 per ADM-8435; source table to confirm | Phase 2 |
| Institutions as entities | Deferred | Free text; needs cleaning and ROR mapping | After cleanup |
| Participant-level data | Excluded | Outside Layer 1 | Future governed layer |
| Non-study folders (All Entities) | Excluded | Containers, not entities any competency question asks about | If a question needs folder structure |
| Tables and views (All Entities) | Excluded | Portal infrastructure, not research outputs | Not planned |
| RecordSets (All Entities) | Excluded | Curator metadata records, not data files | Not planned |
| Dataset rows in All Entities | Excluded from All Entities | Sourced from Portal - Datasets instead, to avoid duplicate or conflicting nodes | — |

## 9. Competency question coverage in v0.1

| # | Question (short) | v0.1 | Note |
|---|---|---|---|
| 1 | Released studies with [data type] from humans | Yes | |
| 2 | Studies with openly accessible data | Yes (currently returns nothing) | `Open` is the value; no study or dataset is currently `Open`. Live study values: Controlled 10, Registered 9, Not Applicable 7, empty 1 |
| 3 | [Specimen type] from more than one species | Yes | |
| 4 | Related studies sharing a project or grant | Partly | Project yes; grant deferred |
| 5 | Publications from the project behind a study | Partly | Answerable via study `publicationIds`, not via grant |
| 6 | Tools from ELITE program projects | No | Tools deferred to Phase 2 |
| 7 | Data types per species | Yes | |
| 8 | Programs with studies marked Data Pending | Yes | Live `dataStatus` values are `Available` (24) and `Data Pending` (5) |
| 9 | Studies, publications, tools per project | Partly | Tools deferred |
| 10 | Institutions generating data for 2+ projects | Yes, as literals | Uses `dataGenerationInstitution` (the only institution column on study rows); results depend on consistent spelling |
| 11 | People on grants funding [data type] studies | No | Person and Grant deferred |

## 10. Open questions

**Resolved 2026-11-16** against the live views: the entity type column is `type`; studies are `type = folder` + `metadataType = study`; files link to studies by `studyKey`; datasets by `studyId` or `studyKey`; `relatedStudies` and `publicationIds` hold Synapse IDs; `dataStatus` values are `Available` and `Data Pending`; the only institution column is `dataGenerationInstitution`.

Also resolved: syn51407023 is the current Publications table; the open `dataRestriction` value is `Open` (none in use yet); files belong to released studies when their `studyKey` matches a study with `isReleased = true`; Programs are keyed by `program` (`ELITE`, `CDCP`).

Also decided 2026-11-17:

- **Namespace:** ELITE uses its own `elite:` namespace.
- **File scope:** file-level metadata is in scope for all released studies, including controlled-access ones.
- **Projects:** the Projects `shortName` for the NECS project is now `NECS`. `CDCP` is a program, so it stays out of Projects; records with `project = CDCP` need an NA-style value and get no project link meanwhile.

**Still open:**

1. Records whose `project` is still `NECS APOE` (at least two datasets as of 2026-11-16) no longer match a Projects row now that the `shortName` is `NECS`. Update them to `NECS`, or they'll be logged as unmatched and get no project link.

## 11. Relationship to the LinkML conversion (ADM-8587)

This ontology is hand-written for v0.1, matching NF and ALS. Property local names mirror ELITE data model attribute names where one exists, so once the LinkML model covers these entities, `ontology.ttl` can be generated from it without renaming graph terms. The SSSOM lookups are shared: the LinkML epic's term mapping tickets write them, and the graph pipeline reads them.
