# Mondo Source Ingest — Reference Specification

This document is the reference specification for the `mondo-source-ingest` agent skill. It covers the target pipeline architecture, source format handling rules, worked examples, and the rationale behind key design decisions.

---

## Overview

Each Mondo ingest source lives in its own independent repository. That repository:

1. Acquires the source data in whatever form it is published
2. Transforms it into a validated LinkML instance (`source.linkml.yml`) — the primary contract
3. Derives an OWL representation (`source.linkml.owl`) for OWL-native consumers
4. Publishes both as GitHub Release artefacts

The agent skill scaffolds this repository and walks the user through each stage interactively. It operates in two modes:

- **Scaffold mode** — creates the repo boilerplate once at initialisation
- **Dialogue mode** — guides the user through source inspection, field mapping, extractor implementation, validation, and release

At each step the agent explains what it is doing and what a successful outcome looks like. It uses the worked examples below to illustrate patterns for common source types.

---

## Phase 1: Source intake questions

The skill begins by asking a short set of questions to classify the source.

### Question 1 — Where does the source come from?

> Where is the upstream data for this source? Provide a URL, API endpoint, or file path.

The answer determines whether the source is:
- a stable static URL (OWL, OBO, TTL, TSV, JSON file)
- a versioned API (BioPortal, WHO ICD API, OncoTree API)
- a GitHub release from another repo

### Question 2 — What format is the source in?

> What format is the source data in? (OWL/OBO/RDF, JSON/API response, TSV/CSV, other)

This determines the extraction path:

| Source format | Extraction approach |
|---|---|
| OWL / OBO / TTL | Load with RDF library; optionally ROBOT-preprocess if structure is broken |
| JSON API / file | Parse and flatten directly into model instances |
| TSV / CSV | Read rows and map fields |

### Question 3 — Does access require authentication?

> Does downloading or querying this source require API keys or credentials?

If yes, the skill scaffolds an `env/.env` file and `.env.example`, and wires the key into the acquisition script.

Examples:
- ICD10CM uses `BIOPORTAL_API_KEY` (BioPortal submission API)
- ICD10WHO uses `CLIENT_ID` and `CLIENT_SECRET` (WHO OAuth2)
- OncoTree uses a public API — no key needed

### Question 4 — Does the source have versioned releases?

> Does the source publish versioned snapshots, or is it a live/latest endpoint?

This determines whether to build a version-resolver script (like `scripts/get_latest_bioportal.py` in icd10cm) or just pin to the latest endpoint.

---

## Phase 2: Scaffold the repo

The skill creates the following directory and file structure:

```
<source-name>/
├── .github/
│   └── workflows/
│       ├── build.yml        # runs on PRs that touch source files
│       └── release.yml      # runs on main, creates dated tag, uploads artefacts
├── config/
│   ├── property-map.sssom.tsv   # maps source property IRIs to Mondo IRIs
│   └── properties.txt           # allowlist of Mondo-approved properties (OWL sources only)
├── docs/
│   └── plan.md              # this repo's own pipeline plan (generated from dialogue)
├── env/
│   ├── .env.example         # template for required env vars
│   └── .env                 # gitignored, actual credentials
├── linkml/
│   └── mondo_source_schema.yaml  # shared ontology schema
├── scripts/
│   ├── acquire.py           # downloads/fetches the source
│   └── extract.py           # ETL: populates LinkML model classes from source
├── src/
│   └── <source_name>/
│       └── datamodel.py     # generated Pydantic classes from LinkML schema
├── tmp/                     # gitignored build artefacts
├── justfile                 # task runner: build, validate, release
├── pyproject.toml           # uv-managed dependencies
└── uv.lock
```

### Tooling setup

- **`uv`** manages the Python environment and dependencies
- **`justfile`** coordinates tasks (replaces Makefile for Python work; a Makefile is also scaffolded for ROBOT steps if the source is OWL)
- **`linkml`**, **`linkml-owl`**, **`pydantic`** are declared as dependencies in `pyproject.toml`

The skill generates all of the above, then confirms with the user before proceeding.

---

## Phase 3: Understand the shared schema

The skill explains the shared ontology schema and asks the user to confirm it is sufficient or identify any gaps.

The base schema (`linkml/mondo_source_schema.yaml`) defines:

```yaml
classes:
  OntologyDocument:
    tree_root: true
    class_uri: owl:Ontology
    slots: [title, version, terms]

  OntologyTerm:
    class_uri: owl:Class
    slots: [id, label, definition, exact_synonyms, related_synonyms,
            narrow_synonyms, broad_synonyms, close_synonyms, parents, is_root]
```

The skill generates Pydantic classes from this schema:

```bash
gen-pydantic linkml/mondo_source_schema.yaml > src/<source_name>/datamodel.py
```

This produces typed Python classes like `OntologyDocument` and `OntologyTerm`. The extractor will use these directly.

---

## Phase 4: Source analysis dialogue

This is the most important phase. The skill inspects the source with the user and identifies how to map source fields to schema slots.

The agent does not guess silently. It shows the user what it sees in the source data and asks for confirmation at each mapping.

### Step 4.1 — Acquire and inspect source sample

The skill downloads a small sample and shows the user its structure.

**Example (OncoTree API):**
```json
{
  "code": "GNOS",
  "name": "Glioma, NOS",
  "parent": "DIFG",
  "externalReferences": {
    "NCI": ["C3058"],
    "UMLS": ["C0017638"]
  }
}
```

**Example (ICD10CM OWL):** the skill runs a small SPARQL probe to show sample triples.

**Example (ICD10WHO TTL):** the skill parses a few triples and shows the prefix structure, class IRIs, and properties used.

### Step 4.2 — Identify labels

> I can see the source uses the field `name` for the primary label. Does this match `rdfs:label` in Mondo? Or is there a preferred label field?

The agent maps the identified field to the `label` slot. For OWL sources it looks for `rdfs:label`. For JSON it asks explicitly.

### Step 4.3 — Identify definitions

> Does the source provide definitions? I'm looking for a text field that describes what each term means — this maps to `obo:IAO_0000115` in Mondo.

Sometimes this is obvious (`definition`, `description`). Sometimes it is absent. The skill notes this and proceeds without requiring it.

### Step 4.4 — Identify synonyms

> Next I want to identify if the source provides any synonyms. This is not always obvious.
>
> **Exact synonyms** (`oboInOwl:hasExactSynonym`) are alternative names that mean exactly the same thing — for example, a disease might have an acronym or an older official name.
>
> **Related synonyms** are terms that are close but not exact — colloquial names, lay terms, spelling variants.
>
> In ICD10CM, the label itself often doubles as an exact synonym and we generate one from the label using a SPARQL update (`exact_syn_from_label.ru`). In ICD10WHO, the same convention applies. In OncoTree, there are no synonyms at all in the API — just the name.
>
> Does this source provide any synonym-like fields? If so, what are they called?

The agent documents the decision and wires it into the extractor.

### Step 4.5 — Identify hierarchy

> How does the source represent parent-child relationships?
>
> - In ICD10CM and ICD10WHO, `rdfs:subClassOf` is already present in the OWL.
> - In OncoTree JSON, each term has a `"parent"` field containing the code of its parent.
> - In ORDO (Orphanet), hierarchy uses part-of restrictions rather than direct subClassOf — so ROBOT/SPARQL must rewrite those first before extraction is possible.
>
> What does the hierarchy look like in this source?

### Step 4.6 — Identify obsolete terms

> Does the source have deprecated or obsolete terms? If so, how are they marked?
>
> - ICD10CM and ICD10WHO use `owl:deprecated true`.
> - OncoTree has `revocations` and `precursors` fields, which require a second pass to reconstruct which old terms were replaced by which new ones.
>
> I'll skip deprecated terms by default in the extracted YAML. Should they be included?

### Step 4.7 — Identify cross-references and mappings

> Does the source contain cross-references or mappings to other ontologies?
>
> For example, OncoTree `externalReferences` contains NCI and UMLS codes, which map to `skos:exactMatch` triples in the output.
>
> These can be included in the extractor output or handled separately in an SSSOM mapping file.

### Step 4.8 — Identify source-specific structural issues (OWL only)

For OWL sources, the skill runs a structural probe and reports anomalies.

> I found that this source uses part-of restrictions in the class hierarchy rather than direct `rdfs:subClassOf`. This is the same situation as ORDO (Orphanet). I'll add a ROBOT/SPARQL preprocessing step that rewrites part-of into subClassOf before extraction.
>
> Is that correct?

Known structural patterns and how they are handled:

| Pattern | Source | Fix |
|---|---|---|
| part-of restrictions instead of subClassOf | ORDO (older) | `ordo-construct-subclass-from-part-of.ru` before extraction |
| illegal punning (property used as both annotation + object property) | OMIM | `fix_illegal_punning_omim.ru` before extraction |
| nested annotation reification | ORDO | `fix_complex_reification_ordo.ru` before extraction |
| self-revocation in replacement lists | OncoTree | skip self-referencing entries |
| split/merge replacements | OncoTree | 1→1 use `IAO:0100001`; 1→many use `oboInOwl:consider` |
| `owl:deprecated` as plain string literal (not `xsd:boolean`) | ORDO | Use `FILTER(str(?dep) = "true")` in all SPARQL that tests deprecation, not `?cls owl:deprecated true` |

**Critical `config/properties.txt` rule for OWL sources:** The ROBOT `remove --select complement --select properties` step strips every annotation property not in the allowlist — including built-ins. Always include the following in `properties.txt` or the extractor will find no labels:
```
http://www.w3.org/2000/01/rdf-schema#label
http://www.w3.org/2002/07/owl#deprecated
```

For non-OWL sources, this step is skipped entirely.

---

## Phase 5: Build the extractor

Based on the analysis dialogue, the skill writes `scripts/extract.py`. This module:

1. Loads the source
2. Iterates over terms
3. Creates `OntologyTerm` instances using the generated `datamodel.py` classes
4. Assembles them into an `OntologyDocument`
5. Serializes to `source.linkml.yml`

The skill shows a sketch of the extractor and asks the user to confirm the field mapping before generating the full implementation.

**Example sketch (OncoTree):**

```python
from src.oncotree.datamodel import OntologyDocument, OntologyTerm

def extract(api_data: list[dict]) -> OntologyDocument:
    terms = []
    for item in api_data:
        term = OntologyTerm(
            id=f"ONCOTREE:{item['code']}",
            label=item["name"],
            parents=[f"ONCOTREE:{item['parent']}"] if item.get("parent") else [],
            is_root=not bool(item.get("parent")),
        )
        terms.append(term)
    return OntologyDocument(
        title="OncoTree",
        terms=terms,
    )
```

**Example sketch (ICD10CM OWL):**

```python
from src.icd10cm.datamodel import OntologyDocument, OntologyTerm
import pyoxigraph as px

def extract(store: px.Store) -> OntologyDocument:
    terms = []
    for row in store.query(CLASSES_QUERY):
        term = OntologyTerm(
            id=curie_from_iri(row["cls"].value),
            label=row["label"].value,
            definition=row.get("def", {}).get("value"),
            parents=[curie_from_iri(p) for p in get_parents(store, row["cls"].value)],
            exact_synonyms=get_synonyms(store, row["cls"].value, EXACT_SYN),
        )
        terms.append(term)
    return OntologyDocument(title="ICD10CM", terms=terms)
```

---

## Phase 6: Validate

The skill runs `linkml-validate` on the produced YAML:

```bash
just validate
# or
python -m linkml.validator.cli \
  -s linkml/mondo_source_schema.yaml \
  -C OntologyDocument \
  source.linkml.yml
```

If validation fails, the agent:
1. Shows which terms or fields failed
2. Explains the likely cause (missing required field, wrong type, invalid IRI format)
3. Suggests a fix in the extractor
4. Re-runs until validation passes

Expected outcome at end of this phase:
- `source.linkml.yml` validates without errors
- Term counts are logged and reviewed with the user

---

## Phase 7: Derive OWL

The skill converts the validated YAML to OWL using `linkml-convert`:

```bash
just data2owl
# or
python -m linkml_owl.dumpers.owl_dumper \
  -s linkml/mondo_source_schema.yaml \
  source.linkml.yml \
  -o source.linkml.owl
```

Notes:
- The derived OWL is for OWL-native consumers only (ontology browsers, ROBOT pipelines)
- It is not the primary ingest artefact — `source.linkml.yml` is
- Known limitation: `linkml-owl` emits OWL Functional format; ROBOT may not load it cleanly in all cases (observed in `icd10cm` and `icd10who`)

---

## Phase 8: Release

The skill wires up a GitHub Actions workflow that:

1. Runs on push to `main` and on a weekly schedule
2. Installs dependencies with `uv sync`
3. Runs `just build` (acquire → extract → validate → data2owl)
4. Creates a dated release tag (`vYYYYMMDD-<run_number>`)
5. Uploads `source.linkml.yml` and `source.linkml.owl` as release assets

Expected release artefacts:
- `source.linkml.yml` — primary ingest-facing product
- `source.linkml.owl` — derived OWL for OWL consumers

For sources that also produce SSSOM mappings (like OncoTree), the workflow also uploads `source.sssom.tsv`.

---

## Phase 9: Verify

The skill guides the user through a verification pass before treating the release as trusted.

### Verification checklist

| Check | How |
|---|---|
| Term count matches source | Compare term count in YAML to source count from download step |
| Required fields populated | Confirm `label` is non-null for all terms; `id` is a valid CURIE |
| Hierarchy is connected | Check that all parent references resolve to known terms |
| Validation passes | `linkml-validate` exits 0 |
| OWL can be loaded | Spot-check `source.linkml.owl` with ROBOT or Protégé |
| Diff against mondo-ingest (if migrating) | `robot diff` between this repo's OWL and the reference from mondo-ingest releases |

---

## Worked examples

### Example A: ICD10CM (OWL from API, versioned)

| Aspect | Detail |
|---|---|
| Source format | OWL (BioPortal API) |
| Auth | `BIOPORTAL_API_KEY` |
| Version resolver | `scripts/get_latest_bioportal.py` writes `DOWNLOAD_URL` + `VERSION_IRI` |
| OWL preprocessing | None structurally required; ROBOT used to remove imports and junk properties |
| Extractor input | ROBOT-preprocessed mirror OWL |
| Synonym strategy | No synonyms in source; exact synonyms generated from label |
| Key source-specific issue | Submission ID was previously hardcoded; resolver script decouples this |
| Release artefacts | `icd10cm.linkml.yml`, `icd10cm.linkml.owl` |

Extractor note: because the source is OWL, the extractor queries it with SPARQL or a SPARQL-capable library (e.g. `pyoxigraph`).

---

### Example B: ICD10WHO (TTL from WHO API)

| Aspect | Detail |
|---|---|
| Source format | Turtle (generated by a separate ingest library from WHO OAuth2 API) |
| Auth | `CLIENT_ID` + `CLIENT_SECRET` (WHO ICD API) |
| Version resolver | None — ingest library handles it; output is `icd10who.ttl` |
| OWL preprocessing | ROBOT `merge` + `normalize` to produce mirror; same SPARQL fixes as ICD10CM |
| Extractor input | ROBOT-preprocessed mirror OWL |
| Synonym strategy | Same as ICD10CM — exact synonym generated from label |
| Key source-specific issue | Term IRIs are WHO `browse10` URLs, not OBO PURLs; need CURIE mapping |
| Release artefacts | `icd10who.linkml.yml`, `icd10who.linkml.owl` |

---

### Example C: OncoTree (JSON API, non-OWL)

| Aspect | Detail |
|---|---|
| Source format | JSON (public REST API or file, two shapes: flat list or nested dict) |
| Auth | None |
| Version resolver | `--version` flag selects a named release; default is latest |
| OWL preprocessing | None — source is not OWL |
| Extractor input | JSON parsed directly into `OntologyTerm` instances |
| Synonym strategy | No synonyms in source |
| Key source-specific issue 1 | Two source shapes (API vs file) require a dual parser |
| Key source-specific issue 2 | Revocations and precursors require a second pass to construct obsolete terms with `IAO:0100001` or `oboInOwl:consider` |
| Key source-specific issue 3 | Self-revocation (a term revoking itself) must be silently ignored |
| Release artefacts | `oncotree.linkml.yml`, `oncotree.linkml.owl`, `oncotree.sssom.tsv` |

For OncoTree, the agent-driven source analysis dialogue naturally discovers the revocation/precursor pattern when the user inspects the JSON shape and asks: "What does the `revocations` field mean?" The agent walks through the 1→1, 1→many, and many→1 replacement cases before writing the extractor.

---

## Source format decision tree

The skill uses this logic to pick the extraction path:

```
Is the source OWL/OBO/TTL/RDF?
├── Yes
│   ├── Does the hierarchy use rdfs:subClassOf directly?
│   │   ├── Yes → ETL directly from OWL
│   │   └── No (e.g. part-of restrictions like ORDO)
│   │       → SPARQL preprocessing first, then ETL
│   └── Are there structural issues (punning, reification)?
│       └── Yes → SPARQL fix, then ETL
└── No (JSON, TSV, CSV, API)
    └── ETL directly from native format
        └── Does the source have a nested structure?
            ├── Yes → flatten first (OncoTree file format)
            └── No → parse directly (OncoTree API format)
```

---

## What the agent figures out automatically

The agent does not require the user to know these things in advance. It discovers them during the source analysis phase:

- Whether exact synonyms exist or must be derived from labels
- Whether the hierarchy needs structural preprocessing
- Whether revocations/precursors require a second extraction pass
- Whether the source has a versioning API or a stable URL
- Whether the source has authentication requirements
- Which source fields map to which schema slots
- Whether the output YAML passes validation; and if not, which extractor changes fix it

The user only needs to:
- Provide the source location
- Confirm or reject field mappings the agent proposes
- Review the final YAML before release

---

## Justfile targets

The skill generates a `justfile` with at least these targets:

| Target | What it does |
|---|---|
| `just build` | acquire → extract → validate → data2owl |
| `just acquire` | fetch/download source only |
| `just extract` | run extractor → `source.linkml.yml` |
| `just validate` | `linkml-validate` on `source.linkml.yml` |
| `just data2owl` | convert YAML → `source.linkml.owl` |
| `just release` | tag and upload release artefacts |
| `just iterate` | extract → validate loop (tight feedback cycle) |

---

## Open design questions

The following are unresolved decisions that affect multiple source repos and should be addressed at the Mondo ingest architecture level:

1. **Schema location** — Should the shared `mondo_source_schema.yaml` live in a dedicated shared repo (pulled in at build time) or be vendored into each source repo independently?
2. **Datamodel files** — Should generated Pydantic classes be committed to the repo, or regenerated from the schema at build time?
3. **Cross-references** — When a source provides mappings to external ontologies (e.g. NCI, UMLS codes in OncoTree), should they be embedded in `source.linkml.yml` or published separately as `source.sssom.tsv`?
4. **Migration timeline** — At what point should `mondo-ingest` switch from consuming OWL release artefacts to consuming `source.linkml.yml` directly?
5. **SPARQL preprocessing scope** — For OWL sources, which structural fixes must remain as pre-extraction ROBOT/SPARQL steps, and which can be absorbed into extractor logic instead?
