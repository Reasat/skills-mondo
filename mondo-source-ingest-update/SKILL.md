---
name: mondo-source-ingest-update
description: >-
  Drives an existing Mondo source ingest repo through the full mondo-source-ingest workflow
  to completion. Use when the user wants to update, modernize, or finish an ingest repo and
  mentions this skill or "mondo-source-ingest-update".
---

# Mondo Source Ingest — Update

Work in an **existing** Mondo source ingest repository (not a blank scaffold). Otherwise, do **exactly** what **[mondo-source-ingest](../mondo-source-ingest/SKILL.md)** says: follow **every phase through Phase 9** to completion — intake where still unknown, scaffold gaps, schema/datamodel as needed, source analysis and scripts, validate and iterate, derive OWL if applicable, wire CI/release, run `verify.py` and record release notes and incidents.

Skip re-asking intake or re-probing mappings **only** when that information is already settled in the repo’s `docs/plan.md` and the user is not changing upstream behaviour; if anything material is missing or stale, fill it in by following the base skill’s steps.

**`reports/`:** When the pipeline publishes OWL (including **non-OWL → `linkml-owl`**), scaffold or verify **`reports/`** and **`just reports` / `make reports`** per the base skill section **`reports/` folder (ROBOT QC)** — not optional just because the *source* is non-OWL.

**External-release / mondo-ingest wget (when modernizing toward ICD10WHO-style):** Follow the base skill **Phase 8 — Mondo-ingest external-release contract**:

- Released `<source>.owl` must be **RDF/XML** (`robot convert` after `linkml-owl`; functional dump stays in `tmp/`)
- Ship the wget bundle with **basenames matching mondo-ingest destinations** (`.db`, `mirror-<source>.owl`, `mirror_signature-<source>.tsv`, `component_signature-<source>.tsv`, SSSOM, metrics)
- Build `semsql` `.db` **in this source repo**, not in mondo-ingest
- Document `releases/latest` race and wget freshness tradeoffs in `docs/plan.md`

Reference implementation: [icd10who](https://github.com/Reasat/icd10who) (see its README workflow + release assets).

All guardrails and conventions in **mondo-source-ingest** apply unchanged.
