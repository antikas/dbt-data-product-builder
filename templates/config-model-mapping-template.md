# Config-Model Mapping: {pipeline_name}

**Generated from:** `{config_file_path}`
**Generated at:** {timestamp}
**Pipeline version:** {pipeline_version}

---

## Pipeline Overview

| Property | Value |
|----------|-------|
| Name | {pipeline_name} |
| Product Type | {product_type} |
| Owner | {owner} |
| Domain | {domain} |
| Strategy | {incremental or full_refresh} |

---

## Step-to-Model Mapping

| Config Step | Pattern | Model / Artefact | Materialisation |
|-------------|---------|-------------------|----------------|
| Step 1 | batch_transfer | `_sources.yml` (source definitions) | n/a |
| Step 2 | data_validation - pre | Tests in `_sources.yml` or `_stg__models.yml` | n/a |
| Step 3 | schema_transform | `int_{entity}_mapped.sql` | view |
| Step 4 | calculated_fields | Inline CTEs in `int_{entity}_mapped.sql` | (inline) |
| Step 5 | data_enrichment | `int_{entity}_enriched.sql` | view |
| Step 6 | data_filtering | WHERE clause in `int_{entity}_enriched.sql` | (inline) |
| Step 7 | data_curation | `int_{entity}_curated.sql` | view |
| Step 8 | data_aggregation | `int_{entity}_aggregated.sql` | table |
| Step 9 | data_validation - post | Tests in `_fnd_{entity}__models.yml` | n/a |
| Step 10 | data_contracts | Contract in `_fnd_{entity}__models.yml` | n/a |
| Step 11 | lineage_capture | This document + dbt native DAG | n/a |
| Step 12 | metadata_capture | `meta:` tags in schema.yml files | n/a |
| Step 13 | schema_publish | `access:` in `_fnd_{entity}__models.yml` | n/a |
| Step 14 | data_publish | Build guidance in model comments | n/a |
| **Final** | - | `fnd_{entity}.sql` | table |

*Rows for steps not present in the config are omitted.*

---

## Field Lineage

### {target_field_1}

| Stage | Field Name | Model | Transformation |
|-------|-----------|-------|---------------|
| Source | `{source_field}` | `source('{schema}', '{table}')` | Raw value |
| Schema Transform | `{target_field}` | `int_{entity}_mapped` | `CAST({source_field} AS {type})` |
| Enrichment | `{target_field}` | `int_{entity}_enriched` | Pass-through |
| Foundation | `{target_field}` | `fnd_{entity}` | Final output |

### {target_field_2} (calculated)

| Stage | Field Name | Model | Transformation |
|-------|-----------|-------|---------------|
| Calculated | `{field_name}` | `int_{entity}_mapped` (CTE) | `{expression}` |
| Foundation | `{field_name}` | `fnd_{entity}` | Final output |

### {target_field_3} (enriched)

| Stage | Field Name | Model | Transformation |
|-------|-----------|-------|---------------|
| Source | `{source_key}` | `int_{entity}_mapped` | Join key |
| Reference | `{ref_field}` | `ref('{reference_model}')` | Lookup value |
| Enrichment | `{target_field}` | `int_{entity}_enriched` | LEFT JOIN on `{join_key}` |
| Foundation | `{target_field}` | `fnd_{entity}` | Final output |

*Repeat for each field in the contract.*

---

## Test Mapping

| Config Rule | Test Type | Location | Severity |
|------------|-----------|----------|----------|
| {rule_description} | `{dbt_test_type}` | `{schema_yml_file}` | {error/warn} |

---

## Notes

- Fields marked with `(inline)` are CTEs within the parent model, not separate files.
- dbt native lineage (the DAG) provides model-level dependencies automatically.
- This document provides **field-level** lineage that dbt's DAG does not capture.
- Regenerate this document whenever the config changes by re-running the generator.
