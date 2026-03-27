# Config-Model Mapping: foundation.fund_master

**Generated from:** `examples/investment/config.yml`
**Pipeline version:** 1.0.0
**Owner:** investment-data-team
**Domain:** investments
**Strategy:** incremental (merge on fund_id)

---

## Step-to-Model Mapping

| Config Step | Pattern | Model / Artefact | Materialisation |
|-------------|---------|-------------------|----------------|
| 1 | batch_transfer | `_sources.yml` (gp_quarterly_report, admin_fund_data) | n/a |
| 2 | data_validation - pre | Tests in `_sources.yml` (not_null, accepted_values, range) | n/a |
| 3a | schema_transform | `int_fund_master_mapped_gp.sql` | view |
| 3b | schema_transform | `int_fund_master_mapped_admin.sql` | view |
| 4 | calculated_fields | Inline CTEs in mapped_gp (is_active, tvpi, dpi, unfunded) and mapped_admin (fund_age_years) | (inline) |
| 5 | data_enrichment | Inline CTEs in `int_fund_master_curated.sql` (strategy_lookup, currency_lookup) | (inline) |
| 6 | data_filtering | WHERE clause - flag column in `int_fund_master_curated.sql` | (inline) |
| 7 | data_curation | `int_fund_master_curated.sql` | view |
| 8 | data_validation - post | Tests in `_fnd_fund_master__models.yml` - 2 singular tests | n/a |
| 9 | data_contracts | Contract in `_fnd_fund_master__models.yml` | n/a |
| 10 | lineage_capture | This document + dbt native DAG | n/a |
| 11 | metadata_capture | `meta:` tags in `_fnd_fund_master__models.yml` | n/a |
| 12 | schema_publish | `access: public`, `latest_version: 1` in `_fnd_fund_master__models.yml` | n/a |
| 13 | data_publish | Build guidance in `fnd_fund_master.sql` header + grants + post_hook | n/a |
| **Final** | - | `fnd_fund_master.sql` | incremental |

---

## Field Lineage

### fund_id

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source (GP) | `fund_id` | `source('gp_quarterly_report', 'gp_quarterly_reports')` | Raw |
| Source (Admin) | `fund_id` | `source('admin_fund_data', 'administrator_fund_records')` | Raw |
| Transform (GP) | `fund_id` | `int_fund_master_mapped_gp` | `CAST(fund_id AS varchar)` |
| Transform (Admin) | `fund_id` | `int_fund_master_mapped_admin` | `CAST(fund_id AS varchar)` |
| Curation | `fund_id` | `int_fund_master_curated` | `COALESCE(gp.fund_id, admin.fund_id)` |
| Foundation | `fund_id` | `fnd_fund_master` | Pass-through |

### fund_name

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `fund_name` | GP source | Raw |
| Transform | `fund_name` | `int_fund_master_mapped_gp` | `CAST(fund_name AS varchar)` |
| Curation | `fund_name` | `int_fund_master_curated` | `COALESCE(gp.fund_name, admin.legal_name)` - GP preferred |
| Foundation | `fund_name` | `fnd_fund_master` | Pass-through |

### legal_name

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `legal_name` | Admin source | Raw |
| Transform | `legal_name` | `int_fund_master_mapped_admin` | `CAST(legal_name AS varchar)` |
| Curation | `legal_name` | `int_fund_master_curated` | `COALESCE(admin.legal_name, gp.fund_name)` - Admin preferred (field override) |
| Foundation | `legal_name` | `fnd_fund_master` | Pass-through |

### strategy_name (enriched)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `strategy_code` | Admin source | Raw |
| Transform | `strategy_code` | `int_fund_master_mapped_admin` | `CAST(strategy_code AS varchar)` |
| Curation | `strategy_code` | `int_fund_master_curated` | Pass-through from admin |
| Enrichment | `strategy_name` | `int_fund_master_curated` (CTE: enriched_strategy) | LEFT JOIN `dim_fund_strategies` ON strategy_code |
| Foundation | `strategy_name` | `fnd_fund_master` | Pass-through |

### tvpi (calculated)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `nav_mm`, `distributed_mm`, `called_capital_mm` | GP source | Raw |
| Transform | `nav_mm`, `distributed_mm`, `called_capital_mm` | `int_fund_master_mapped_gp` | CAST to decimal(18,4) |
| Calculated | `tvpi` | `int_fund_master_mapped_gp` (CTE: calculated) | `ROUND((nav_mm + distributed_mm) / NULLIF(called_capital_mm, 0), 4)` |
| Curation | `tvpi` | `int_fund_master_curated` | Pass-through from GP |
| Foundation | `tvpi` | `fnd_fund_master` | Pass-through |

### has_data_quality_flag (filter flag)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Filter | `has_data_quality_flag` | `int_fund_master_curated` (CTE: filtered) | `CASE WHEN NOT (committed_capital_mm > 0) THEN true ELSE false END` |
| Foundation | `has_data_quality_flag` | `fnd_fund_master` | Pass-through |

### usd_fx_rate (enriched with default)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `currency` | GP/Admin source | Raw |
| Enrichment | `usd_fx_rate` | `int_fund_master_curated` (CTE: enriched_currency) | LEFT JOIN `dim_currencies` ON currency = currency_code; COALESCE(ref.usd_fx_rate, 1.0) |
| Foundation | `usd_fx_rate` | `fnd_fund_master` | Pass-through |

---

## Test Mapping

| Config Rule | Test Type | Location | Severity |
|------------|-----------|----------|----------|
| fund_id completeness (pre) | `not_null` | `_sources.yml` (both sources) | error |
| fund_id uniqueness (pre) | `unique_combination_of_columns` | `_sources.yml` (GP source) | error |
| vintage_year range (pre) | `accepted_range` | `_sources.yml` (GP source) | error |
| strategy accepted_values (pre) | `accepted_values` | `_sources.yml` (GP source) | warn |
| currency accepted_values (pre) | `accepted_values` | `_sources.yml` (GP source) | error |
| nav_mm range (pre) | `accepted_range` | `_sources.yml` (GP source) | warn |
| fund_id uniqueness (post) | `unique` | `_fnd_fund_master__models.yml` | error |
| fund_id completeness (post) | `not_null` | `_fnd_fund_master__models.yml` | error |
| fund_name completeness (post) | `not_null` (threshold 0.95) | `_fnd_fund_master__models.yml` | warn |
| committed_capital range (post) | `accepted_range` | `_fnd_fund_master__models.yml` | error |
| TVPI non-negative (post) | Singular test | `tests/singular/assert_fund_master_tvpi_non_negative.sql` | error |
| Called <= committed (post) | Singular test | `tests/singular/assert_fund_master_called_le_committed.sql` | error |

---

## Notes

- Two schema_transforms (one per source) feed into a single data_curation step. This is the multi-source FDP pattern.
- Calculated fields are split: financial metrics (tvpi, dpi, unfunded) on the GP transform; fund_age_years on the admin transform (because it needs inception_date).
- Enrichment and filtering are inlined in the curation model to avoid unnecessary intermediate models.
- The quarantine model captures records failing error-severity post-validation rules before they reach the foundation model.
