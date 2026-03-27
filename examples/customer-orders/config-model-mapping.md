# Config-Model Mapping: curated.customer

**Generated from:** `examples/customer-orders/config.yml`
**Pipeline version:** 2.1.0
**Owner:** customer-domain
**Domain:** customer
**Strategy:** full_refresh

---

## Step-to-Model Mapping

| Config Step | Pattern | Model / Artefact | Materialisation |
|-------------|---------|-------------------|----------------|
| 1 | batch_transfer | `_sources.yml` (crm_customer, billing_customer) | n/a |
| 2 | data_validation - pre | Tests in `_sources.yml` | n/a |
| 3a | schema_transform | `int_customer_mapped_crm.sql` | view |
| 3b | schema_transform | `int_customer_mapped_billing.sql` | view |
| 4 | calculated_fields | Inline CTEs in `int_customer_mapped_crm.sql` | (inline) |
| 5 | data_enrichment | Inline CTEs in `int_customer_curated.sql` | (inline) |
| 6 | data_filtering | WHERE clause in `int_customer_curated.sql` | (inline) |
| 7 | data_curation | `int_customer_curated.sql` | view |
| 8 | data_validation - post | Tests in `_cur_customer__models.yml` | n/a |
| 9 | data_contracts | Contract in `_cur_customer__models.yml` | n/a |
| 10 | lineage_capture | This document + dbt native DAG | n/a |
| 11 | metadata_capture | `meta:` tags in `_cur_customer__models.yml` | n/a |
| 12 | schema_publish | `access: protected` in `_cur_customer__models.yml` | n/a |
| 13 | data_publish | Build guidance in `cur_customer.sql` header | n/a |
| **Final** | - | `cur_customer.sql` | table |

---

## Field Lineage (selected fields)

### full_name (calculated)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `first_name`, `last_name` | CRM source | Raw |
| Calculated | `full_name` | `int_customer_mapped_crm` (CTE: calculated) | `TRIM(first_name \|\| ' ' \|\| last_name)` |
| Curation | `full_name` | `int_customer_curated` | CRM preferred (default survivorship) |
| Curated | `full_name` | `cur_customer` | Pass-through |

### customer_segment (enriched)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `customer_type` | CRM source | Raw |
| Enrichment | `customer_segment` | `int_customer_curated` (CTE: enriched_segment) | LEFT JOIN `dim_customer_segments` ON customer_type = type_code; COALESCE(segment_name, 'Unclassified') |
| Curated | `customer_segment` | `cur_customer` | Pass-through |

### credit_limit (billing source preferred)

| Stage | Field | Model | Transformation |
|-------|-------|-------|---------------|
| Source | `credit_limit` | Billing source | Raw |
| Transform | `credit_limit` | `int_customer_mapped_billing` | `CAST(credit_limit AS decimal(12,2))` |
| Curation | `credit_limit` | `int_customer_curated` | Billing preferred (field override, rule: source_priority) |
| Curated | `credit_limit` | `cur_customer` | Pass-through |

---

## Test Mapping

| Config Rule | Test Type | Location | Severity |
|------------|-----------|----------|----------|
| customer_id completeness (pre) | `not_null` | `_sources.yml` (both) | error |
| customer_id uniqueness (pre) | `unique` | `_sources.yml` (both) | error |
| email format (pre) | `expect_column_values_to_match_regex` | `_sources.yml` (CRM) | warn |
| customer_type accepted_values (pre) | `accepted_values` | `_sources.yml` (CRM) | error |
| customer_id uniqueness (post) | `unique` | `_cur_customer__models.yml` | error |
| customer_id completeness (post) | `not_null` | `_cur_customer__models.yml` | error |
| customer_type accepted_values (post) | `accepted_values` | `_cur_customer__models.yml` | error |
