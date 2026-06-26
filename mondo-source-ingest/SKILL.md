---
name: mondo-source-ingest
description: Scaffolds a new Mondo disease ontology source ingest repo and guides the user through building the ETL pipeline in a dialogue-style workflow. Use when a user wants to create a new Mondo source repo, onboard a new ingest source, or set up a new source preprocessing pipeline for Mondo.
---

# Mondo Source Ingest

You are helping a user scaffold and build a new Mondo source ingest repo. Work interactively — ask one question at a time, inspect source data before proposing mappings, and confirm decisions with the user before writing code.

Before starting, read [plan.md](plan.md) for the full architecture rationale, source format decision tree, and worked examples (ICD10CM, ICD10WHO, OncoTree).

---

## Phase 1: Intake

Ask these four questions sequentially. Do not ask them all at once.

**Q1 — Source location:**
> Where is the upstream data? Provide a URL, API endpoint, or file path.

**Q2 — Source format:** (ask after Q1)
> What format is the source in? (OWL/OBO/RDF, JSON API, JSON file, TSV/CSV, other)

**Q3 — Authentication:** (ask after Q2)
> Does accessing this source require API keys or credentials?

**Q4 — Versioning:** (ask after Q3)
> Does the source publish versioned snapshots, or is it a live/latest endpoint?

After Q4, summarise what you understood and confirm before scaffolding.

---

## Phase 2: Scaffold

Create the repo structure. Confirm the directory name with the user first.

```
<source-name>/
├── .github/workflows/
│   ├── build.yml
│   └── release.yml
├── config/
│   ├── property-map.sssom.tsv
│   └── properties.txt          # OWL sources only
├── docs/
│   ├── plan.md                 # pipeline logic: source, field mappings, design decisions, ID scheme
│   ├── pipeline_incidents.md   # unanticipated events, errors, deviations and how they were resolved
│   └── release_notes.md        # ontology stats + Phase 9 verification results for each release
├── env/
│   ├── .env.example
│   └── .env                    # gitignored
├── linkml/
│   └── mondo_source_schema.yaml
├── reports/                    # ROBOT QC output (see “`reports/` folder” below); committed
├── scripts/
│   ├── acquire.py              # fetch/download source (all source types)
│   ├── transform.py            # OWL sources: ROBOT-output OWL → YAML
│   ├── extract.py              # non-OWL sources: raw JSON/TSV/API → YAML
│   ├── verify.py               # structural checks on the produced YAML
│   └── resolve_version.py      # optional: versioned sources only
├── sparql/                     # OWL: `*.ru` (ROBOT updates). Optional `*.sparql` SELECT for `robot query` / `reports/`
│   └── *.ru                    # SPARQL update queries applied via ROBOT
├── src/<source_name>/
│   └── datamodel.py            # generated from schema
├── tmp/                        # gitignored
├── Makefile                    # OWL sources: generic pipeline (sourced from project.Makefile)
├── project.Makefile            # OWL sources: source-specific rules and download URLs
├── odk.sh                      # OWL sources: Docker wrapper (./odk.sh make all)
├── justfile                    # non-OWL sources
├── pyproject.toml
├── README.md
└── uv.lock
```

**`reports/` folder (ROBOT QC):**

- **Purpose:** Committed **QC metrics** from ROBOT on whatever release OWL you ship (extended `robot measure` JSON, optional `robot query` TSVs such as `top-level-counts.tsv` from `sparql/count_classes_by_top_level.sparql`).
- **OWL-first sources (`Makefile`):** `make reports` runs `robot measure` + SPARQL counts against **mirror**, **transformed**, and **final** OWL (three top-level TSVs + `metrics.json` — see `make reports` below).
- **Non-OWL sources (`justfile`) that run `linkml-owl`:** You still **publish** `<source>.linkml.owl`. Add a **`just reports`** (or equivalent) that runs the **same** ROBOT commands against **that single OWL** (typically one `metrics.json` and one `top-level-counts.tsv`). **Do not** treat “non-OWL” as “no `reports/`” — the label means the *source* is not OWL; the **derived OWL** is still amenable to ROBOT QC.
- **YAML-only:** If a repo intentionally releases **no** OWL at all, omit `reports/` (nothing for ROBOT to measure).
- **Phase 4.8:** “Skip for non-OWL” applies only to the **pre-extraction OWL probe** (SPARQL *update* chain on raw upstream OWL). It does **not** exempt you from **post‑`data2owl`** reports when you emit a OWL file.

Note: exact script filenames (`transform.py` vs `extract.py`) are confirmed during Phase 4 once the source format and processing steps are known.

**`.gitignore`** — scaffold this at repo creation. Build artefacts are gitignored because they are uploaded as GitHub Release assets, not committed:

```gitignore
# Credentials — never commit
.env
env/.env

# Build artefacts — uploaded as GitHub Release assets
# OWL sources: <source>.yaml (YAML) and <source>.owl (final LinkML-derived OWL)
# Non-OWL sources: <source>.linkml.yaml and <source>.linkml.owl
<source>.owl
<source>.yaml
<source>.linkml.yaml
<source>.linkml.owl

# Build intermediates
tmp/

# Python
*.pyc
__pycache__/
*.egg-info/

# Virtual env — uv.lock IS committed for reproducible CI builds
.venv/
```

**`pyproject.toml`** dependencies: `linkml`, `pydantic`, `PyYAML`, `rdflib`. Add `linkml-owl` for all source types — OWL sources call it via `make build` to derive the final `<source>.owl` from the YAML; non-OWL sources call it directly via `just data2owl`. Note that for OWL sources `linkml-owl` is installed at the pinned version by `make dependencies` (not from `pyproject.toml`) because the required version differs from what `uv sync` would resolve. Add `pyoxigraph` if SPARQL querying inside the Python script is needed.

**Build tool differs by source type:**

**OWL sources — `Makefile` + `project.Makefile`:**

Use a two-file split. The generic `Makefile` contains the reusable pipeline (always sourced via `include project.Makefile` at the bottom). The `project.Makefile` contains the source-specific download URL and ROBOT preprocessing chain. All file paths derive from a single `SOURCE ?= <source-name>` variable. Key derived variables:

```makefile
SOURCE      ?= <source-name>
RAW_OWL     := tmp/$(SOURCE)_raw.owl
MIRROR_OWL  := tmp/mirror-$(SOURCE).owl
OUTPUT_OWL  := tmp/transformed-$(SOURCE).owl   # ROBOT-processed intermediate
OUTPUT_OWL_LINKML := $(SOURCE).owl              # final LinkML-derived OWL (top-level)
YAML_OUT    := $(SOURCE).yaml
MIR         ?= true   # set MIR=false to skip re-downloading: make MIR=false build
```

> **Open design question:** `MIRROR_OWL` and `OUTPUT_OWL` are currently in the generic `Makefile`. For sources that have no OWL intermediate at all (e.g. an API-based pipeline that produces YAML directly without a ROBOT step), these variables have no meaning in the generic file. Consider moving them to `project.Makefile` on a case-by-case basis — keep them in the generic file when every variant of the pipeline will use them, move them to `project.Makefile` when they are source-specific.

Core `Makefile` targets:
- `make all` — `build` + `reports`
- `make mirror` — download raw source OWL to `tmp/`
- `make build` — full pipeline: ROBOT preprocessing → transform → validate → linkml-owl
- `make reports` — `robot measure` (extended JSON metrics) + `count_classes_by_top_level.sparql` across mirror/transformed/final OWL; produces `reports/metrics.json`, `reports/mirror-top-level-counts.tsv`, `reports/transformed-top-level-counts.tsv`, `reports/top-level-counts.tsv`
- `make robot-plugins` — copies ROBOT plugin JARs from `/tools/robot-plugins/` (ODK) or local `plugins/` into `tmp/plugins/`; exports `ROBOT_PLUGINS_DIRECTORY`
- `make dependencies` — installs `linkml-owl==0.5.0` plus bleeding-edge `linkml` and `linkml-runtime` from the `linkml/linkml` monorepo (required for the inlining bug fix — see Known Issues)
- `make update-schema` — downloads schema from `SCHEMA_URL`
- `make clean` — removes `tmp/`, `reports/`, build artefacts
- `make help` — prints usage

All Python commands run as bare `python` (no `uv run` wrapper) because the pipeline runs inside the ODK Docker container where dependencies are pre-installed via `make dependencies`.

**`odk.sh`** — Docker wrapper for local execution:
```sh
#!/bin/sh
IMAGE=${IMAGE:-odkfull}
docker run -v "$PWD/:/work" -w /work \
  -e ROBOT_JAVA_ARGS="-Xmx20G" -e JAVA_OPTS="-Xmx20G" \
  --rm -ti obolibrary/$IMAGE "$@"
```
Usage: `./odk.sh make all` or `./odk.sh make MIR=false build`.

**Non-OWL sources — `justfile`:**
- `just acquire` — fetch source
- `just extract` — source → `source.linkml.yaml`
- `just validate` — `python -m linkml.validator.cli -s linkml/mondo_source_schema.yaml -C OntologyDocument source.linkml.yaml`
- `just build` — full pipeline end-to-end
- `just iterate` — extract → validate loop only (tight feedback, skips acquire)
- `just data2owl` — YAML → `<source>.linkml.owl` (if not folded into `build`)
- `just reports` — **recommended** when `build` emits OWL: `robot measure` + optional SPARQL counts → `reports/` (see **`reports/` folder** above)
- `just release` — tag and upload

If auth is needed, scaffold `env/.env.example` and load credentials from `.env` in `acquire.py`. Load with `load_dotenv()` first, then `os.getenv()` — this means CI can pass credentials as environment variables without needing the `.env` file present.

If the source is versioned, scaffold a `scripts/resolve_version.py` that writes the resolved URL and version IRI to `.env`.

**For API traversal sources** (sources where `acquire.py` must iterate node-by-node through an API rather than download a single file):
- **Use an explicit queue or stack — never recursion.** Python's default 1,000-frame recursion limit silently truncates recursive traversals mid-run without raising an error. The choice between BFS (queue, `pop(0)`) and iterative DFS (stack, `pop()`) doesn't matter — both are safe. What matters is that the "call stack" is a plain Python list in heap memory, not the interpreter's call stack.
- **Token refresh:** if the API uses short-lived OAuth tokens (~1 hour), implement proactive refresh (e.g. every 55 minutes) inside the traversal loop, plus a reactive catch on 401 responses. A single token fetch at startup is insufficient for long traversals.
- **Cache every node response** to `tmp/cache/<encoded-uri>/response.json` as the traversal runs. If the run is interrupted, the next run reads from cache rather than re-fetching.
- **Wire `actions/cache` in CI** to persist `tmp/cache/` between workflow runs, keyed on a hash of `acquire.py`. The first CI run pays the full traversal cost; all subsequent runs restore the cache and finish in seconds:
  ```yaml
  - name: Restore API node cache
    uses: actions/cache@v4
    with:
      path: tmp/cache
      key: <source>-api-cache-${{ hashFiles('scripts/acquire.py') }}
      restore-keys: |
        <source>-api-cache-
  ```

For **live/latest endpoints** (no explicit version in the URL), scaffold a `resolve_latest_url()` function in `acquire.py` that scrapes the source's download page and extracts the current filename via regex. Always prefer dynamic resolution over hardcoding a version-specific URL — a hardcoded URL will silently fetch a stale file once the source publishes a new version.

**`README.md`** — keep it minimal. The detailed pipeline rationale lives in `docs/plan.md`. The README should only contain:

```markdown
# <source-name>

<One sentence description>.

## Setup

<Auth steps if needed, e.g.:>
1. Register at <auth URL> to get API credentials
2. Copy `env/.env.example` → `env/.env` and fill in the required variables
3. Install dependencies: `uv sync`

## Run

```bash
# OWL sources (run inside ODK Docker via odk.sh, or locally with ROBOT installed):
./odk.sh make all          # full pipeline: mirror + build + reports
./odk.sh make MIR=false build   # skip re-downloading

# Non-OWL sources:
just acquire
just build
```

## Outputs

| File | Description |
|---|---|
| `<source>.yaml` | Primary artefact for Mondo ingest |
| `<source>.owl` | Final OWL (LinkML-derived; OWL sources also have `tmp/transformed-<source>.owl` as ROBOT intermediate) |
| `reports/` | ROBOT QC (`robot measure` + optional SPARQL); **OWL-first** (three OWL inputs) **or** **non–OWL + `linkml-owl`** (single derived OWL) — see **`reports/` folder** above |

## Docs

| Doc | Contents |
|---|---|
| [`docs/plan.md`](docs/plan.md) | Pipeline architecture, field mappings, ID scheme |
| [`docs/release_notes.md`](docs/release_notes.md) | Ontology stats and verification results per release |
| [`docs/pipeline_incidents.md`](docs/pipeline_incidents.md) | Pipeline incidents: errors, deviations, resolutions |
```

If `acquire` is slow (e.g. API traversal), add a note in the `make acquire` line so the user knows it is expected behaviour, e.g. `# ~2.5 hrs, cached after first run`.

---

## Phase 3: Schema and datamodel

Copy `mondo_source_schema.yaml` into `linkml/`. The base schema below is the correct working template (v0.4.0) — do not simplify it. It has been validated against `linkml-owl` and produces correct OWL output.

```yaml
id: https://w3id.org/monarch-initiative/mondo-source-schema
name: mondo_source_schema
version: 0.4.0

prefixes:
  linkml:    https://w3id.org/linkml/
  mondo_src: https://w3id.org/monarch-initiative/mondo-source-schema/
  rdfs:      http://www.w3.org/2000/01/rdf-schema#
  skos:      http://www.w3.org/2004/02/skos/core#
  dcterms:   http://purl.org/dc/terms/
  obo:       http://purl.obolibrary.org/obo/
  oboInOwl:  http://www.geneontology.org/formats/oboInOwl#
  owl:       http://www.w3.org/2002/07/owl#
  MONDO:     http://purl.obolibrary.org/obo/mondo#
  # ADD source-specific prefix here, e.g.:
  # ICD10WHO: https://icd.who.int/browse10/2019/en#/
  # Orphanet: http://www.orpha.net/ORDO/Orphanet_

imports:
  - linkml:types

default_prefix: mondo_src
default_range: string


enums:

  SynonymTypeEnum:
    description: Types of synonyms used in Mondo source ingests.
    permissible_values:
      omim_included:
        meaning: MONDO:omim_included
      generated_from_label:
        meaning: MONDO:GENERATED_FROM_LABEL
      generated:
        meaning: MONDO:GENERATED
      omim_formerly:
        meaning: MONDO:omim_formerly
      abbreviation:
        meaning: MONDO:ABBREVIATION


classes:

  OntologyDocument:
    tree_root: true
    class_uri: owl:Ontology
    slots:
      - title
      - version
      - terms

  Synonym:
    description: A synonym value with an optional type annotation.
    slots:
      - synonym_text
      - synonym_type

  OntologyTerm:
    class_uri: owl:Class
    slots:
      - id
      - label
      - definition
      - exact_synonyms
      - related_synonyms
      - narrow_synonyms
      - broad_synonyms
      - skos_exact_match
      - parents
      - deprecated
    # is_root is computed internally in transform.py (to decide whether a term has parents)
    # but is NEVER written to YAML or declared in the schema — it would emit spurious OWL.
    slot_usage:
      exact_synonyms:
        annotations:
          owl.template: |-
            {% for s in exact_synonyms %}
            AnnotationAssertion({% if s.synonym_type %}Annotation(oboInOwl:hasSynonymType {{s.synonym_type.meaning}}) {% endif %}oboInOwl:hasExactSynonym {{id}} "{{s.synonym_text|replace('"', '\\"')}}")
            {% endfor %}
      related_synonyms:
        annotations:
          owl.template: |-
            {% for s in related_synonyms %}
            AnnotationAssertion({% if s.synonym_type %}Annotation(oboInOwl:hasSynonymType {{s.synonym_type.meaning}}) {% endif %}oboInOwl:hasRelatedSynonym {{id}} "{{s.synonym_text|replace('"', '\\"')}}")
            {% endfor %}
      narrow_synonyms:
        annotations:
          owl.template: |-
            {% for s in narrow_synonyms %}
            AnnotationAssertion({% if s.synonym_type %}Annotation(oboInOwl:hasSynonymType {{s.synonym_type.meaning}}) {% endif %}oboInOwl:hasNarrowSynonym {{id}} "{{s.synonym_text|replace('"', '\\"')}}")
            {% endfor %}
      broad_synonyms:
        annotations:
          owl.template: |-
            {% for s in broad_synonyms %}
            AnnotationAssertion({% if s.synonym_type %}Annotation(oboInOwl:hasSynonymType {{s.synonym_type.meaning}}) {% endif %}oboInOwl:hasBroadSynonym {{id}} "{{s.synonym_text|replace('"', '\\"')}}")
            {% endfor %}


slots:

  title:
    slot_uri: rdfs:label
    required: true

  version:
    slot_uri: owl:versionInfo
    required: true

  terms:
    range: OntologyTerm
    multivalued: true
    inlined_as_list: true
    required: true

  synonym_text:
    required: true

  synonym_type:
    range: SynonymTypeEnum

  id:
    identifier: true
    slot_uri: dcterms:identifier
    range: uriorcurie        # must be uriorcurie — plain string breaks linkml-owl IRI resolution
    required: true

  label:
    slot_uri: rdfs:label
    required: true
    annotations:
      owl: AnnotationAssertion

  definition:
    slot_uri: obo:IAO_0000115
    annotations:
      owl: AnnotationAssertion

  exact_synonyms:
    slot_uri: oboInOwl:hasExactSynonym
    multivalued: true
    range: Synonym
    inlined_as_list: true

  related_synonyms:
    slot_uri: oboInOwl:hasRelatedSynonym
    multivalued: true
    range: Synonym
    inlined_as_list: true

  narrow_synonyms:
    slot_uri: oboInOwl:hasNarrowSynonym
    multivalued: true
    range: Synonym
    inlined_as_list: true

  broad_synonyms:
    slot_uri: oboInOwl:hasBroadSynonym
    multivalued: true
    range: Synonym
    inlined_as_list: true

  skos_exact_match:
    slot_uri: skos:exactMatch
    multivalued: true
    range: uriorcurie
    annotations:
      owl: AnnotationAssertion

  parents:
    slot_uri: rdfs:subClassOf
    range: OntologyTerm
    multivalued: true
    annotations:
      owl: SubClassOf          # SubClassOf, not AnnotationAssertion

  deprecated:
    slot_uri: owl:deprecated
    range: boolean
    # NO ifabsent here — ifabsent: "false" emits owl:deprecated false on every non-deprecated term
    annotations:
      owl: AnnotationAssertion
```

**Adding object property slots (OWL sources with RO/BFO restrictions):**

If the source uses OWL object property restrictions (e.g. ORDO uses part-of and material-basis relations), add these to `OntologyTerm.slots` and the top-level `slots:` block. Object properties use `range: OntologyTerm` and `owl: ObjectSomeValuesFrom` — not `AnnotationAssertion`:

```yaml
# In OntologyTerm slots: list
      - has_material_basis_in_germline_mutation_in
      - has_material_basis_in_somatic_mutation_in
      - has_material_basis_in
      - part_of
      - has_part

# In top-level slots:
  has_material_basis_in_germline_mutation_in:
    slot_uri: RO:0004001
    multivalued: true
    range: OntologyTerm
    annotations:
      owl: ObjectSomeValuesFrom

  part_of:
    slot_uri: BFO:0000050
    multivalued: true
    range: OntologyTerm
    annotations:
      owl: ObjectSomeValuesFrom
```

Add `RO: http://purl.obolibrary.org/obo/RO_` and `BFO: http://purl.obolibrary.org/obo/BFO_` to `prefixes:` when these slots are used.

**Critical `linkml-owl` rules — violations produce a silent empty OWL file (no error):**
1. **Use top-level `slots:`, not inline `attributes:`** — `linkml-owl` only emits axioms for slots declared at the top level with `annotations: owl:`.
2. **Every annotation property slot must have `annotations: owl: AnnotationAssertion`** — without it, `linkml-owl` silently omits that slot from the derived OWL.
3. **`parents` must use `annotations: owl: SubClassOf`** — not `AnnotationAssertion`.
4. **`id` must have `range: uriorcurie`** — plain `string` prevents `linkml-owl` from resolving CURIEs to IRIs. All class declarations will be missing.
5. **The source IRI namespace must be declared in `prefixes:`** — if `ICD10WHO:A00.0` is an `id` value but `ICD10WHO:` is not in the prefix map, `linkml-owl` cannot expand it and silently skips the class.
6. **When adding source-specific extra slots**, always include `annotations: owl: AnnotationAssertion` (or `owl: ObjectSomeValuesFrom` for object properties) on them too.
7. **Do not add `ifabsent: "false"` to the `deprecated` slot.** This causes `owl:deprecated false` to be emitted as an annotation axiom on every non-deprecated term, polluting the output OWL. Leave `deprecated` without any `ifabsent`.
8. **`is_root` must never appear in the schema or YAML output.** It is internal state in `transform.py` (used to decide whether a term has parents). If it appears in the schema, `linkml-owl` emits it as a spurious annotation on every root term.
9. **Synonym slots must use `range: Synonym` + `inlined_as_list: true` + `owl.template` in `OntologyTerm.slot_usage`.** The older pattern of `multivalued: true` plain strings with `annotations: owl: AnnotationAssertion` is incompatible with synonym type annotation. `linkml-owl`'s `AnnotationAssertion` mode cannot reach into inlined objects, so the Jinja `owl.template` is required. Transform scripts must emit synonyms as objects — `{"synonym_text": "..."}` — not plain strings.

**Diagnosing a silent failure:** if `linkml-owl` produces a file under ~1 KB containing only the ontology header and zero `AnnotationAssertion` lines, one of the above rules has been violated. Check them in order.

Then generate the Pydantic datamodel:
```bash
gen-pydantic linkml/mondo_source_schema.yaml > src/<source_name>/datamodel.py
```

Ask the user if they need any additional slots before generating.

**Schema slot type warning:** The `in_subsets` slot has `range: uriorcurie`. Do not put plain text strings (e.g. source-internal classification labels like "Clinical subtype") into this slot — `linkml-owl` will reject them with `ValueError: X is not a valid URI or CURIE`. If the source has free-text classification values, either drop them or add a dedicated `range: string` slot for them.

---

## Phase 4: Source analysis dialogue

Download a small sample. Show the user what you see. Then ask about each mapping one at a time.

**4.1 — Labels:**
> I can see [field name] appears to be the primary label. Should this map to `rdfs:label`?

For OWL sources, look for `rdfs:label`. For JSON, identify the name field.

**4.2 — Definitions:**
> Does this source have definitions — a text field explaining what each term means? This would map to `obo:IAO_0000115`. If not, that slot will remain empty.

**4.3 — Synonyms:**
> Next, I want to identify synonyms. This is not always obvious.
>
> Exact synonyms are alternative names that mean exactly the same thing (acronyms, official alternate names). Related synonyms are close but not exact (colloquial names, spelling variants).
>
> In ICD10CM and ICD10WHO, there are no explicit synonyms in the source — we generate an exact synonym from the label itself. In OncoTree, there are no synonyms at all.
>
> Does this source provide any synonym-like fields? If not, should I generate exact synonyms from labels?

**4.4 — Hierarchy:**
> How does the source represent parent-child relationships?
>
> - OWL sources usually have `rdfs:subClassOf` directly.
> - JSON sources often have a `parent` or `parent_code` field.
> - ORDO uses part-of restrictions instead of subClassOf — that requires a SPARQL rewrite before extraction.
>
> What does the hierarchy look like here?

**4.5 — Obsolete terms:**
> Does the source have deprecated or obsolete terms? If so, how are they marked?
>
> ICD10CM/ICD10WHO use `owl:deprecated true`. OncoTree uses `revocations` and `precursors` fields, which require a second pass — each revoked code becomes an obsolete class pointing to its replacement via `IAO:0100001`, or `oboInOwl:consider` if there are multiple replacements. Note: ignore self-revocations.
>
> Should I include obsolete terms in the output?

**4.5b — Non-disease term exclusion:**
> Some sources include top-level grouping nodes whose direct children are not diseases — they are gene/locus type classifications, administrative groupings, or metadata artefacts (e.g. ORDO's `Orphanet_C010` "genetic material" parent, whose children include `gene with protein product`, `non coding RNA`, `disorder-associated locus`). These terms are incorrectly typed as `owl:Class` in the source OWL but must not appear as disease classes in the output.
>
> Ask: are there any top-level grouping terms in this source whose subtrees should be excluded from the ingest? If so, build an exclusion set in `transform.py` and apply it both when collecting terms and when collecting parents (so no retained term references an excluded term via `rdfs:subClassOf`).

**4.6 — Cross-references and mappings:**
> Does the source contain cross-references to other ontologies (e.g. NCI, UMLS, OMIM)?
>
> If so, do **not** carry them through as `oboInOwl:hasDbXref` — that predicate should not appear in the final OWL. Instead, collect them into `skos_exact_match` (the `skos:exactMatch` slot in the schema). This applies to both `oboInOwl:hasDbXref` values and source-internal notation codes (e.g. ORPHA codes from `skos:notation`).
>
> In `transform.py`, merge all xrefs into `skos_exact_match` after normalising their CURIE prefixes. The `sparql/fix_xref_prefixes.ru` SPARQL update (see Phase 4.8) normalises prefixes on the OWL side before extraction; the Python side should deduplicate and sort the result.
>
> Mappings may additionally be published as an SSSOM file if the project requires it.

**4.7 — IRI namespace and CURIE scheme:**
> What IRI namespace does the source use for its class identifiers?
>
> - OBO sources use `http://purl.obolibrary.org/obo/<PREFIX>_<code>` — these resolve cleanly to standard CURIEs.
> - Some sources use their own namespaces (e.g. ICD10WHO uses `https://icd.who.int/browse10/2019/en#/`).
>
> If the source IRI namespace is not an OBO PURL, declare a custom prefix in the schema's `prefixes:` block and document the CURIE scheme in `docs/plan.md`. The prefix must be declared in the schema for `linkml-owl` to resolve it — an undeclared prefix causes `linkml-owl` to silently skip all class declarations.

**4.8 — OWL structural issues (OWL sources only):**

Run a SPARQL probe. Report any of these patterns and ask for confirmation before adding a ROBOT preprocessing step:

| Pattern | Seen in | Fix |
|---|---|---|
| part-of restrictions instead of subClassOf | ORDO | SPARQL rewrite before extraction |
| illegal punning | OMIM | SPARQL fix before extraction |
| nested annotation reification | ORDO | SPARQL fix before extraction |
| Non-standard CURIE prefixes in `hasDbXref` | ORDO, many sources | `sparql/fix_xref_prefixes.ru` (include by default) |

**`sparql/fix_xref_prefixes.ru` — include for every OWL source.** This file is ported from `mondo-ingest` and normalises the most common bad CURIE prefixes in `oboInOwl:hasDbXref` values before the Python transform reads them:

```sparql
PREFIX oboInOwl: <http://www.geneontology.org/formats/oboInOwl#>

DELETE { ?s oboInOwl:hasDbXref ?old }
INSERT { ?s oboInOwl:hasDbXref ?new }
WHERE {
  ?s oboInOwl:hasDbXref ?old .
  BIND(
    IF(STRSTARTS(STR(?old), "ICD-11:"),  CONCAT("ICD11:",   SUBSTR(STR(?old), 8)),
    IF(STRSTARTS(STR(?old), "ICD-10:"),  CONCAT("ICD10:",   SUBSTR(STR(?old), 8)),
    IF(STRSTARTS(STR(?old), "MeSH:"),    CONCAT("MESH:",    SUBSTR(STR(?old), 6)),
    IF(STRSTARTS(STR(?old), "OMIM:PS"),  CONCAT("OMIMPS:",  SUBSTR(STR(?old), 8)),
       STR(?old)))))
    AS ?new
  )
  FILTER(?old != ?new)
}
```

Also strip non-breaking spaces (`U+00A0`) from xref values if the source is known to emit them. Apply this update in the `project.Makefile` ROBOT chain before the property filter step.

**`sparql/count_classes_by_top_level.sparql` — scaffold for every OWL source.** The `make reports` target runs this query against the mirror, transformed, and final OWL to count descendant classes under the source's top-level grouping terms. For **non-OWL** sources, reuse the same query file against the **single** `linkml-owl` output in `just reports` (one TSV). Template:

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX owl:  <http://www.w3.org/2002/07/owl#>

SELECT ?topLevel (COUNT(DISTINCT ?child) AS ?count)
WHERE {
  VALUES ?topLevel {
    # REPLACE with the actual top-level IRIs for this source, e.g.:
    # <http://www.orpha.net/ORDO/Orphanet_557493>   # Disorder
    # <http://www.orpha.net/ORDO/Orphanet_557492>   # Group of disorders
    # <http://www.orpha.net/ORDO/Orphanet_557494>   # Subtype of a disorder
  }
  ?child rdfs:subClassOf+ ?topLevel .
  ?child a owl:Class .
}
GROUP BY ?topLevel
```

Ask the user for the top-level grouping IRIs and fill them in during Phase 4.

For non-OWL sources, **skip this Phase 4.8 probe** (there is no raw upstream OWL to preprocess). **Still** add `sparql/count_classes_by_top_level.sparql` (filled with IRIs from the **derived** ontology) and **run it in `just reports`** after `data2owl` — see **`reports/` folder** in Phase 2.

**4.9 — Term count cross-check:**

If a prior version of this source exists anywhere (committed file, BioPortal, another repo), compare term counts before treating a discrepancy as an error. Document the difference and its cause in `docs/pipeline_incidents.md`. Known cause: the old `monarch-initiative/icd10who` TTL had 4,894 terms due to a Python recursion limit truncating the traversal — the correct full count is 12,597.

---

## Phase 5: Write the processing scripts

The pipeline differs by source type. Confirm the approach with the user before writing any code.

---

### 5a — OWL sources: ROBOT preprocessing + `transform.py`

OWL sources go through two stages. Stage 1 is ROBOT (invoked from `Makefile`); Stage 2 is Python.

**Stage 1 — ROBOT (via `Makefile`):**

```
make mirror   → robot merge -i tmp/<source>_raw.owl odk:normalize → tmp/mirror-<source>.owl
make build    → robot merge -i tmp/mirror-<source>.owl
                  remove --select imports
                  rename --mappings config/property-map.sssom.tsv
                  query --update sparql/fix_xref_prefixes.ru     (always include)
                  query --update sparql/fix_*.ru                 (source-specific fixes)
                  query --update sparql/exact_syn_from_label.ru  (if needed)
                  remove -T config/properties.txt --select complement --select properties --trim true
                  annotate --ontology-iri <IRI> --version-iri <VERSION_IRI>
                → tmp/transformed-<source>.owl          ← ROBOT intermediate (not the release artefact)
              then Python transform → <source>.yaml
              then linkml-owl → <source>.owl            ← released OWL artefact (top-level)
```

**Output file naming — important:**
- `tmp/transformed-<source>.owl` — ROBOT-preprocessed intermediate; used as input to `transform.py`
- `<source>.owl` — final OWL derived by `linkml-owl` from the YAML; this is the top-level release artefact
- `<source>.yaml` — primary YAML artefact for Mondo ingest

SPARQL update files (`sparql/*.ru`) handle structural issues identified in Phase 4.8. The property allowlist (`config/properties.txt`) must always include `rdfs:label` and `owl:deprecated` at minimum. The `sparql/fix_xref_prefixes.ru` file (see Phase 5a–SPARQL below) should be included for every OWL source.

The source-specific ROBOT preprocessing chain (download URL, SPARQL file list, `robot` invocation) lives in `project.Makefile`, not the generic `Makefile`.

**Stage 2 — `scripts/transform.py`:**

Reads the ROBOT-output `tmp/transformed-<source>.owl` (not the raw acquired file or the mirror) using rdflib. Maps OWL predicates to schema slots and writes `<source>.yaml`.

Show the user a sketch and confirm field mappings before writing the full implementation:

```python
from rdflib import OWL, RDF, RDFS, Graph, Literal, URIRef
from rdflib.namespace import Namespace

OBOINOWL = Namespace("http://www.geneontology.org/formats/oboInOwl#")

def extract_terms(g: Graph) -> list[dict]:
    for iri in sorted(str(s) for s in g.subjects(RDF.type, OWL.Class) if isinstance(s, URIRef)):
        label = g.value(URIRef(iri), RDFS.label)
        if label is None:
            continue
        exact_syns = [
            {"synonym_text": str(o).strip()}
            for o in g.objects(URIRef(iri), OBOINOWL.hasExactSynonym)
            if str(o).strip()
        ]
        # map further slots: definition, parents, skos_exact_match ...
        # is_root is computed here (no parents → root) but NEVER written to the output dict
        yield {"id": curie(iri), "label": str(label), "exact_synonyms": exact_syns, ...}
```

Key rules for `transform.py`:
- **Synonyms are objects**, not plain strings: `{"synonym_text": "..."}` or `{"synonym_text": "...", "synonym_type": "abbreviation"}`. Plain strings will fail schema validation.
- **`is_root` is internal only.** Compute it to decide whether `parents` is empty, but never write it to the output dict.
- **Xrefs → `skos_exact_match`.** Collect `oboInOwl:hasDbXref` values (and any source notation codes) and merge them into `skos_exact_match` as a list of CURIEs. Do not emit a `database_cross_references` key.

---

### 5b — Non-OWL sources: `extract.py`

No ROBOT involved. The extractor reads the raw acquired file (JSON, TSV, API response) directly.

Show the user a sketch and confirm field mappings before writing the full implementation:

```python
from src.<source>.datamodel import OntologyDocument, OntologyTerm

def extract(data) -> OntologyDocument:
    terms = []
    for item in data:
        has_parent = bool(item.get("parent"))
        terms.append(OntologyTerm(
            id=f"PREFIX:{item['code']}",
            label=item["name"],
            parents=[f"PREFIX:{item['parent']}"] if has_parent else [],
            # is_root is intentionally NOT set — compute internally, never write to output
        ))
    return OntologyDocument(title="Source", terms=terms)
```

For sources with revocations (OncoTree-style), implement a second pass for obsolete terms after the main loop.

---

**Robustness rules (apply to both paths):**
- Always strip and skip blank/whitespace-only literal values before adding them to list slots. `linkml-owl` raises `ConstructorError: Empty list elements are not allowed` if a list contains an empty string. Use `val = str(o).strip(); if val: out.append(val)` in every literal-collecting helper.
- When writing SPARQL that filters on `owl:deprecated`, use `FILTER(str(?dep) = "true")` rather than `?cls owl:deprecated true`. Some sources (including ORDO) serialise the deprecated flag as an untyped plain string literal `"true"` rather than `"true"^^xsd:boolean`. The boolean keyword in SPARQL does not match plain literals.
- **Custom YAML dumper for special characters.** Synonym text (and other string values) can contain `,`, `:`, `{`, or `}`. Standard `yaml.dump` may emit these unquoted, causing downstream parsers to misread the value. Implement a `yaml.Dumper` subclass that quotes strings containing these characters, and pass it to `yaml.dump(..., Dumper=QuotingDumper)`.
- **`linkml-runtime` inlining bug with commas.** There is an active bug in `linkml_runtime._normalize_inlined` (`yamlutils.py`) where string values containing commas in `inlined_as_list` slots (i.e. synonym text with commas) cause a `ValueError` during key parsing. The workaround is to install `linkml-runtime` from the `main` branch of the `linkml/linkml` monorepo. Add this to the `dependencies` Makefile target:
  ```makefile
  dependencies:
      pip install --quiet --break-system-packages linkml-owl==0.5.0 \
          "linkml @ git+https://github.com/linkml/linkml.git@main#subdirectory=packages/linkml" \
          "linkml-runtime @ git+https://github.com/linkml/linkml.git@main#subdirectory=packages/linkml_runtime"
  ```
  Document this in `docs/pipeline_incidents.md` for every new repo until the upstream bug is fixed.

---

## Phase 6: Validate and iterate

Run the tight feedback loop (skips the slow download step):

```bash
# OWL sources:
./odk.sh make MIR=false build

# Non-OWL sources:
just iterate
```

If validation fails:
1. Show the user which fields failed and why
2. Propose a specific fix in the extractor / transform
3. Re-run

Do not proceed to Phase 7 until `linkml-validate` exits 0 and term counts look plausible. Log term counts for the user to review.

---

## Phase 7: Derive OWL

This phase applies to **non-OWL sources only**.

For OWL sources, the released OWL artefact is the ROBOT-processed `source.owl` produced in Phase 5a — no further derivation is needed. Skip this phase.

For non-OWL sources (JSON, TSV, API), the source has no OWL representation, so one must be derived from the YAML:

```bash
just data2owl
# python -m linkml_owl.dumpers.owl_dumper -s linkml/mondo_source_schema.yaml -f yaml <source>.linkml.yaml -o <source>.linkml.owl
```

Tell the user: the derived OWL is for OWL-native consumers only. `<source>.linkml.yaml` is the primary contract. Known limitation: `linkml-owl` emits OWL Functional format; ROBOT may not load it cleanly in all cases. If it fails on large datasets (rdflib N3 parser error), document this in `docs/pipeline_incidents.md` and release `<source>.linkml.yaml` only.

**After `data2owl` succeeds:** run **`just reports`** (or equivalent) so **`reports/`** is populated from the derived OWL (`robot measure` + optional SPARQL). Wire that into **CI** (e.g. `build.yml` artifacts or committed metrics) alongside `verify` — **not** into GitHub Release uploads; release assets stay YAML + OWL only (Phase 8 table). If you skip OWL entirely (YAML-only release), skip `reports/` too — see **`reports/` folder** in Phase 2.

---

## Phase 8: Wire CI and release

Generate the two workflow files below. For OWL sources, all steps run inside `obolibrary/odkfull:v1.6` Docker so that ROBOT and the ODK normalize plugin are available without any separate install step. For non-OWL sources (JSON, TSV), the Docker step can be replaced with a plain `uv sync && uv run python ...` step on `ubuntu-latest`.

**Release schedule — ask the user first:** Do not assume the template’s `schedule` / `cron` block. Ask whether they want **automatic releases on an interval** (and what cadence — e.g. weekly, monthly, or none), versus **`workflow_dispatch` only** (manual), versus **push-to-main** triggers alone. Long API traversals, rate limits, or infrequent upstream changes often mean omitting `schedule` or using a looser interval; document the choice in `docs/plan.md`. The example `release.yml` below includes a **default** weekly cron — replace, remove, or keep it according to their answer.

**`.github/workflows/release.yml`**

```yaml
name: Build and release

on:
  workflow_dispatch:
  schedule:
    - cron: "0 0 * * 1"   # weekly, Monday 00:00 UTC
  push:
    branches: [main]
    paths:
      - "Makefile"         # or justfile for non-OWL sources
      - "config/**"
      - "sparql/**"        # OWL sources only
      - "linkml/**"
      - "scripts/**"
      - "pyproject.toml"
      - "uv.lock"

jobs:
  build-and-release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build (ROBOT + LinkML)
        run: |
          docker run --rm \
            -e ROBOT_PLUGINS_DIRECTORY=/tools/robot-plugins \
            -v "$PWD:/work" \
            -w /work \
            obolibrary/odkfull:v1.6 \
            bash -lc "make dependencies && make all"

      - name: Set release tag
        id: version
        run: echo "tag=v$(date +%Y%m%d)" >> "$GITHUB_OUTPUT"

      - name: Create release and upload assets
        if: success() && hashFiles('<source>.yaml') != ''
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.version.outputs.tag }}
          name: Release ${{ steps.version.outputs.tag }}
          files: |
            <source>.yaml
            <source>.owl
          generate_release_notes: true
          draft: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Substitute the actual output filenames. Released artefacts differ by source type:

| Source type | Released YAML | Released OWL |
|---|---|---|
| OWL | `<source>.yaml` | `<source>.owl` (final LinkML-derived OWL, top-level) |
| Non-OWL | `<source>.linkml.yaml` | `<source>.linkml.owl` (linkml-owl derived) |

**GitHub Release assets (`action-gh-release` `files`):** upload **only** the YAML and OWL from the table above. Do **not** attach `reports/*` (or other QC) as release assets — keep `reports/` in-repo (committed or regenerated in CI) and/or as **workflow artifacts** in `build.yml`, not as downloadable release files.

Note: for OWL sources, the ROBOT-preprocessed intermediate (`tmp/transformed-<source>.owl`) is not released — it lives in `tmp/` which is gitignored.

**`.github/workflows/build.yml`**

```yaml
name: Build

on:
  pull_request:
    paths:
      - "Makefile"
      - "config/**"
      - "sparql/**"
      - "linkml/**"
      - "scripts/**"
      - "pyproject.toml"
      - "uv.lock"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build (ROBOT + LinkML)
        run: |
          docker run --rm \
            -e ROBOT_PLUGINS_DIRECTORY=/tools/robot-plugins \
            -v "$PWD:/work" \
            -w /work \
            obolibrary/odkfull:v1.6 \
            bash -lc "make dependencies && make all"
```

**Key implementation notes:**
- `permissions: contents: write` is required on the release job for `softprops/action-gh-release` to create tags and releases.
- `ROBOT_PLUGINS_DIRECTORY=/tools/robot-plugins` makes the ODK normalize plugin available inside the container (its location in `odkfull`). The `make robot-plugins` target copies JARs from there into `tmp/plugins/` before ROBOT runs.
- `hashFiles('<source>.yaml') != ''` guards the release step so a failed build does not create an empty release.
- `make dependencies` installs the pinned `linkml-owl==0.5.0` and bleeding-edge `linkml`/`linkml-runtime` from the monorepo; this must run before `make all` inside the container.
- The `workflow_dispatch` trigger allows manual runs from the GitHub Actions UI without a push.
- Local `ROBOT_PLUGINS_DIRECTORY` will differ from CI (e.g. `/home/<user>/.robot/plugins` locally vs `/tools/robot-plugins` in Docker). The `make robot-plugins` target handles this automatically.

---

## Phase 9: Verify

Scaffold `scripts/verify.py` and run it before the first release. Record results in `docs/release_notes.md`.

**`scripts/verify.py`** automates the structural checks. It must:
- Accept `--yaml <path>` (the produced YAML) and `--expected-version <str>` (optional)
- Check `title` and `version` are present and non-empty
- Check for duplicate term IDs
- Check every term has a non-empty `label`
- Check every `parents` entry resolves to a known term ID in the same file (broken refs = hierarchy error)
- Print a summary (term count, unique IDs, broken parent refs) and exit 0 on PASS, exit 1 on FAIL

Run it:
```bash
# OWL sources:
uv run python scripts/verify.py --yaml <source>.yaml --expected-version <version>

# Non-OWL sources:
uv run python scripts/verify.py --yaml <source>.linkml.yaml --expected-version <version>
```

Add this as a `just verify` / `make verify` target so it can be re-run for every release.

**Full checklist (some checks automated by `verify.py`, some manual):**

| Check | How |
|---|---|
| Title and version present | `verify.py` |
| No duplicate term IDs | `verify.py` |
| `label` non-null for all terms | `verify.py` |
| All `parents` refs resolve | `verify.py` |
| `version` matches upstream release identifier | `verify.py --expected-version` |
| `linkml-validate` exits 0 | `uv run python -m linkml.validator.cli ...` |
| OWL artefact loads in ROBOT / Protégé | manual spot-check |
| `robot diff` vs mondo-ingest reference (if migrating) | manual |
| `reports/` (when OWL is released) | `make reports` or `just reports` — `metrics.json` + optional SPARQL TSVs |

**OWL sources additionally:**
- [ ] `<source>.owl` (final LinkML-derived OWL, top-level) can be loaded by ROBOT or opened in Protégé
- [ ] `tmp/transformed-<source>.owl` (ROBOT-preprocessed intermediate) can be loaded as a sanity check
- [ ] If migrating from mondo-ingest: run `robot diff` between this OWL and the mondo-ingest reference OWL

**Non-OWL sources additionally:**
- [ ] `source.linkml.owl` (linkml-owl output) can be loaded by ROBOT or opened in Protégé
- [ ] If you ship OWL: `reports/` is generated from that OWL (not “non-OWL ⇒ no reports”)

---

## Guardrails

- Never propose field mappings without first showing the user a data sample
- Never write the extractor without showing a sketch and getting confirmation
- Never proceed past validation until it passes
- Do not add SPARQL **update** (`*.ru`) preprocessing on **raw upstream OWL** for non-OWL sources (Phase 4.8). **Post-`data2owl`** `robot measure` / `robot query` for **`reports/`** on the **derived** OWL is separate and recommended when OWL is published
- Do not invent synonym behaviour — ask the user if the source has synonyms or if they should be generated from labels
- Never silently remove or simplify a pipeline step because a tool or plugin appears to be missing. Search the full filesystem, then ask the user where it is before removing anything.
- Generate `docs/plan.md` capturing the pipeline logic that governs this repo: upstream source, field-to-slot mappings, ID scheme, versioning strategy, and key design decisions. This is the canonical reference for anyone maintaining the pipeline.
- Generate `docs/pipeline_incidents.md` recording every unanticipated event that occurred during the session — errors, tool failures, necessary deviations from the standard pipeline, and the exact steps taken to resolve each one.
- Generate `docs/release_notes.md` containing the ontology statistics from the full run (term count, definitions, synonyms, roots, broken refs) together with the Phase 9 verification checklist results. Update this file for every subsequent release.
- **Never substitute a third-party or mirror source for the official upstream.** The acquire step must always fetch from the authoritative publisher (e.g. WHO API, BioPortal official submission, ORPHADATA). Third-party builds (e.g. biopragmatics/obo-db-ingest, OBO Foundry mirrors) may be used for *inspection and prototyping only* — never as the production source. If the official source is slow or requires credentials, scaffold the credentials properly and document the performance impact; do not silently swap to a convenience mirror.
- **Always record the official release identifier in the output.** The `version` field in the produced YAML must match the upstream publisher's versioning scheme (e.g. `2026-01` for WHO ICD-11, submission ID for BioPortal). A date derived from a third-party build timestamp is not an acceptable substitute.
