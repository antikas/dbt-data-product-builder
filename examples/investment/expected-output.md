# Investment Example - Expected Generator Output

**Config:** `foundation.fund_master` v1.0.0
**Generated artefacts:** 7 files

---

## File 1: models/sources/_sources.yml

```yaml
version: 2

sources:
  - name: gp_quarterly_report
    schema: raw
    description: "GP quarterly report extractions - fund-level financial data"
    loaded_at_field: extraction_timestamp
    freshness:
      warn_after:
        count: 2
        period: day
      error_after:
        count: 7
        period: day
    tables:
      - name: gp_quarterly_reports
        description: "GP quarterly report extractions - fund-level financial data"
        columns:
          - name: fund_id
            description: "GP-assigned fund identifier"
            tests:
              - not_null
          - name: fund_name
            description: "Fund name as reported by the GP"
          - name: gp_name
            description: "General partner name"
          - name: strategy
            description: "Investment strategy (Buyout, Growth Equity, etc.)"
            tests:
              - accepted_values:
                  values: ["Buyout", "Growth Equity", "Infrastructure", "Real Estate", "Venture Capital", "Credit", "Secondaries"]
                  severity: warn
          - name: vintage_year
            description: "Fund vintage year"
            tests:
              - dbt_utils.accepted_range:
                  min_value: 1990
                  max_value: 2030
          - name: currency
            description: "Reporting currency (ISO 4217)"
            tests:
              - not_null
              - accepted_values:
                  values: ["USD", "EUR", "GBP", "CHF", "JPY", "AUD", "CAD", "SGD", "AED"]
          - name: committed_capital_mm
            description: "Total committed capital in millions"
          - name: called_capital_mm
            description: "Capital called to date in millions"
          - name: nav_mm
            description: "Net asset value in millions"
            tests:
              - dbt_utils.accepted_range:
                  min_value: 0
                  severity: warn
          - name: distributed_mm
            description: "Distributions to date in millions"
          - name: reporting_quarter
            description: "Reporting period (YYYY-QN format)"
          - name: report_date
            description: "Date of the GP report"
          - name: extraction_timestamp
            description: "When this record was extracted from the PDF"

  - name: admin_fund_data
    schema: raw
    description: "Fund administrator records - legal and structural fund data"
    loaded_at_field: last_updated
    freshness:
      warn_after:
        count: 1
        period: day
      error_after:
        count: 3
        period: day
    tables:
      - name: administrator_fund_records
        description: "Fund administrator records - legal and structural fund data"
        columns:
          - name: fund_id
            description: "Administrator-assigned fund identifier (matches GP fund_id)"
            tests:
              - not_null
          - name: legal_name
            description: "Fund legal name (from fund documents)"
          - name: domicile
            description: "Fund domicile jurisdiction"
          - name: fund_structure
            description: "Legal structure (LP, LLC, etc.)"
          - name: inception_date
            description: "Fund inception date"
          - name: strategy_code
            description: "Strategy classification code"
          - name: currency
            description: "Fund base currency (ISO 4217)"
          - name: committed_capital_mm
            description: "Committed capital per administrator records"
          - name: administrator_name
            description: "Name of the fund administrator"
          - name: last_updated
            description: "Last update timestamp from administrator"
```

---

## File 2: models/intermediate/int_fund_master_mapped_gp.sql

```sql
-- Model: int_fund_master_mapped_gp
-- Pattern: schema_transform
-- Source: gp_quarterly_report
-- Generated from: foundation.fund_master v1.0.0
--
-- Purpose: Map GP quarterly report fields to canonical fund model.
-- Deduplication: latest record per fund_id + reporting_quarter.
-- Materialisation: view (structural mapping only - no heavy computation)

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'investments', 'fund_master']
) }}

with source as (

    select * from {{ source('gp_quarterly_report', 'gp_quarterly_reports') }}

),

deduplicated as (

    select
        *,
        row_number() over (
            partition by fund_id, reporting_quarter
            order by extraction_timestamp desc
        ) as _row_num
    from source

),

cleaned as (

    select * from deduplicated
    where _row_num = 1

),

renamed as (

    select
        -- Mapped fields
        cast(fund_id as varchar) as fund_id,
        cast(fund_name as varchar) as fund_name,
        cast(gp_name as varchar) as gp_name,
        cast(strategy as varchar) as strategy,
        cast(vintage_year as integer) as vintage_year,
        cast(currency as varchar) as currency,
        cast(committed_capital_mm as decimal(18,4)) as committed_capital_mm,
        cast(called_capital_mm as decimal(18,4)) as called_capital_mm,
        cast(nav_mm as decimal(18,4)) as nav_mm,
        cast(distributed_mm as decimal(18,4)) as distributed_mm,
        cast(reporting_quarter as varchar) as reporting_quarter,
        cast(report_date as date) as report_date

        -- Dropped: extraction_timestamp (not required in foundation model)

    from cleaned

),

-- Calculated fields (inline, rule version: 2026Q1, owner: investment-data-team)
calculated as (

    select
        *,
        -- Whether the fund is currently active (has NAV or uncalled capital)
        case when nav_mm > 0 or called_capital_mm < committed_capital_mm then true else false end as is_active,
        -- Total Value to Paid-In ratio (TVPI = (NAV + Distributions) / Called)
        round((nav_mm + distributed_mm) / nullif(called_capital_mm, 0), 4) as tvpi,
        -- Distributions to Paid-In ratio (DPI = Distributions / Called)
        round(distributed_mm / nullif(called_capital_mm, 0), 4) as dpi,
        -- Remaining unfunded commitment in millions
        committed_capital_mm - called_capital_mm as unfunded_commitment_mm
    from renamed

)

select * from calculated
```

---

## File 3: models/intermediate/int_fund_master_mapped_admin.sql

```sql
-- Model: int_fund_master_mapped_admin
-- Pattern: schema_transform
-- Source: admin_fund_data
-- Generated from: foundation.fund_master v1.0.0
--
-- Purpose: Map administrator fund records to canonical fund model.
-- Materialisation: view (structural mapping only)

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'investments', 'fund_master']
) }}

with source as (

    select * from {{ source('admin_fund_data', 'administrator_fund_records') }}

),

renamed as (

    select
        -- Mapped fields
        cast(fund_id as varchar) as fund_id,
        cast(legal_name as varchar) as legal_name,
        cast(domicile as varchar) as domicile,
        cast(fund_structure as varchar) as fund_structure,
        cast(inception_date as date) as inception_date,
        cast(strategy_code as varchar) as strategy_code,
        cast(currency as varchar) as currency,
        cast(committed_capital_mm as decimal(18,4)) as committed_capital_mm,
        cast(administrator_name as varchar) as administrator_name

        -- Dropped: last_updated (not required in foundation model)

    from source

),

-- Calculated fields (inline, rule version: 2026Q1, owner: investment-data-team)
calculated as (

    select
        *,
        -- Fund age in whole years from inception
        date_part('year', age(current_date, inception_date)) as fund_age_years
    from renamed

)

select * from calculated
```

---

## File 4: models/intermediate/int_fund_master_curated.sql

```sql
-- Model: int_fund_master_curated
-- Pattern: data_curation
-- Entity key: fund_id
-- Resolution: deterministic
-- Sources (by priority): 1. int_fund_master_mapped_gp, 2. int_fund_master_mapped_admin
--
-- Purpose: Resolve fund entity across GP and administrator sources into a single
-- golden record. GP data preferred for financial metrics; administrator data
-- preferred for legal details.
--
-- Enrichment: strategy lookup (dim_fund_strategies), currency lookup (dim_currencies)
-- Filtering: exclude test funds, exclude pre-cutover, flag zero commitment

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'investments', 'fund_master']
) }}

with gp as (

    select * from {{ ref('int_fund_master_mapped_gp') }}

),

admin as (

    select * from {{ ref('int_fund_master_mapped_admin') }}

),

-- Entity resolution: deterministic match on fund_id
-- Survivorship: GP preferred (priority 1) unless field override specifies admin
curated as (

    select
        coalesce(gp.fund_id, admin.fund_id) as fund_id,

        -- GP preferred (default survivorship)
        coalesce(gp.fund_name, admin.legal_name) as fund_name,
        gp.gp_name,
        coalesce(gp.strategy, null) as strategy,
        coalesce(gp.vintage_year, null) as vintage_year,
        coalesce(gp.currency, admin.currency) as currency,

        -- Admin preferred (field overrides)
        -- Override: legal_name - priority: [admin, gp], rule: first_non_null
        coalesce(admin.legal_name, gp.fund_name) as legal_name,
        -- Override: domicile - priority: [admin], rule: source_priority
        admin.domicile,
        -- Override: fund_structure - priority: [admin], rule: source_priority
        admin.fund_structure,
        -- Override: inception_date - priority: [admin, gp], rule: first_non_null
        coalesce(admin.inception_date, null) as inception_date,
        -- Override: administrator_name - priority: [admin], rule: source_priority
        admin.administrator_name,
        -- Override: committed_capital_mm - priority: [gp, admin], rule: first_non_null
        coalesce(gp.committed_capital_mm, admin.committed_capital_mm) as committed_capital_mm,

        -- GP only (financial metrics)
        gp.called_capital_mm,
        gp.nav_mm,
        gp.distributed_mm,
        gp.reporting_quarter,
        gp.report_date,

        -- Calculated fields (from GP source)
        gp.is_active,
        gp.tvpi,
        gp.dpi,
        gp.unfunded_commitment_mm,

        -- Calculated fields (from admin source)
        admin.fund_age_years,

        -- Strategy code for enrichment
        admin.strategy_code

    from gp
    full outer join admin on gp.fund_id = admin.fund_id

),

-- Enrichment: strategy lookup
enriched_strategy as (

    select
        curated.*,
        -- Canonical strategy name from reference
        ref_strategy.strategy_name,
        -- Asset class classification
        ref_strategy.asset_class,
        -- High-level strategy grouping
        ref_strategy.strategy_group,
        -- Warn flag: strategy lookup failed
        case when ref_strategy.strategy_code is null then true else false end as strategy_lookup_missing
    from curated
    left join {{ ref('dim_fund_strategies') }} as ref_strategy
        on curated.strategy_code = ref_strategy.strategy_code

),

-- Enrichment: currency lookup
enriched_currency as (

    select
        enriched_strategy.*,
        -- Full currency name (default: 'Unknown' if lookup fails)
        coalesce(ref_currency.currency_name, 'Unknown') as currency_name,
        -- Exchange rate to USD (default: 1.0 if lookup fails)
        coalesce(ref_currency.usd_fx_rate, 1.0) as usd_fx_rate
    from enriched_strategy
    left join {{ ref('dim_currencies') }} as ref_currency
        on enriched_strategy.currency = ref_currency.currency_code

),

-- Filtering
filtered as (

    select
        *,
        -- Flag: funds with zero committed capital
        case
            when not (committed_capital_mm > 0) then 'flag_zero_commitment'
            else null
        end as data_quality_flag_reason,
        case
            when not (committed_capital_mm > 0) then true
            else false
        end as has_data_quality_flag
    from enriched_currency
    where true
        -- Filter: exclude_test_funds - Exclude test/dummy funds created during system setup
        and fund_id not like 'TEST%'
        -- Filter: exclude_pre_cutover - Exclude legacy funds before platform cutover
        and vintage_year >= 2000

)

select * from filtered
```

---

## File 5: models/foundation/fnd_fund_master.sql

```sql
-- Model: fnd_fund_master
-- Foundation Data Product: foundation.fund_master v1.0.0
-- Owner: investment-data-team
-- Domain: investments
-- Description: Canonical fund entity - golden record resolved from GP quarterly reports
--   and fund administrator data. One record per fund, with survivorship rules
--   preferring GP data for financial metrics and administrator data for legal details.
--
-- This model is the published foundation data product.
-- It has an enforced contract - downstream consumers depend on this schema.
-- Do not modify without updating the contract and notifying consumers.
--
-- Build: dbt build --select fnd_fund_master+
-- Test:  dbt test --select fnd_fund_master
-- Docs:  dbt docs generate && dbt docs serve

{{ config(
    materialized='incremental',
    schema='foundation',
    tags=['foundation', 'investments', 'fund_master'],
    contract={'enforced': true},
    unique_key='fund_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns',
    grants={'select': ['analyst_role', 'portfolio_manager_role']},
    post_hook="GRANT SELECT ON {{ this }} TO ROLE analyst_role"
) }}

with source as (

    select * from {{ ref('int_fund_master_curated') }}

),

final as (

    select
        fund_id,
        fund_name,
        legal_name,
        gp_name,
        strategy,
        strategy_name,
        asset_class,
        strategy_group,
        vintage_year,
        currency,
        currency_name,
        domicile,
        fund_structure,
        inception_date,
        administrator_name,
        committed_capital_mm,
        called_capital_mm,
        nav_mm,
        distributed_mm,
        tvpi,
        dpi,
        unfunded_commitment_mm,
        is_active,
        fund_age_years,
        reporting_quarter,
        report_date,
        usd_fx_rate,
        has_data_quality_flag,
        data_quality_flag_reason,
        strategy_lookup_missing
    from source
    {% if is_incremental() %}
    where report_date >= dateadd(month, -6, current_date)
    {% endif %}

)

select * from final
```

---

## File 6: models/intermediate/_int_fund_master__models.yml

```yaml
version: 2

models:
  - name: int_fund_master_mapped_gp
    description: "Schema transform: GP quarterly reports → canonical fund model. Deduplicated by fund_id + reporting_quarter (latest extraction wins)."
    config:
      materialized: view
      schema: intermediate
      tags: ['foundation', 'investments', 'fund_master']
    columns:
      - name: fund_id
        description: "Canonical fund identifier"
        tests:
          - not_null
      - name: fund_name
        description: "Fund name as reported by the GP"
      - name: gp_name
        description: "General partner name"
      - name: strategy
        description: "Investment strategy"
      - name: vintage_year
        description: "Fund vintage year"
      - name: currency
        description: "Reporting currency (ISO 4217)"
      - name: committed_capital_mm
        description: "Total committed capital in millions"
      - name: called_capital_mm
        description: "Capital called to date in millions"
      - name: nav_mm
        description: "Net asset value in millions"
      - name: distributed_mm
        description: "Distributions to date in millions"
      - name: reporting_quarter
        description: "Reporting period (YYYY-QN)"
      - name: report_date
        description: "Date of the GP report"
      - name: is_active
        description: "Whether the fund is currently active"
      - name: tvpi
        description: "Total Value to Paid-In ratio"
      - name: dpi
        description: "Distributions to Paid-In ratio"
      - name: unfunded_commitment_mm
        description: "Remaining unfunded commitment in millions"

  - name: int_fund_master_mapped_admin
    description: "Schema transform: administrator fund records → canonical fund model."
    config:
      materialized: view
      schema: intermediate
      tags: ['foundation', 'investments', 'fund_master']
    columns:
      - name: fund_id
        description: "Canonical fund identifier"
        tests:
          - not_null
      - name: legal_name
        description: "Fund legal name from administrator"
      - name: domicile
        description: "Fund domicile jurisdiction"
      - name: fund_structure
        description: "Legal structure (LP, LLC, etc.)"
      - name: inception_date
        description: "Fund inception date"
      - name: strategy_code
        description: "Strategy classification code"
      - name: currency
        description: "Fund base currency (ISO 4217)"
      - name: committed_capital_mm
        description: "Committed capital per administrator"
      - name: administrator_name
        description: "Fund administrator name"
      - name: fund_age_years
        description: "Fund age in whole years from inception"

  - name: int_fund_master_curated
    description: "Entity resolution: golden record per fund from GP + administrator sources. GP preferred for financial metrics, admin for legal details. Enriched with strategy and currency lookups. Test funds and pre-cutover excluded."
    config:
      materialized: view
      schema: intermediate
      tags: ['foundation', 'investments', 'fund_master']
    columns:
      - name: fund_id
        description: "Canonical fund identifier (entity-resolved)"
        tests:
          - not_null
          - unique
```

---

## File 7: models/foundation/_fnd_fund_master__models.yml

```yaml
version: 2

models:
  - name: fnd_fund_master
    description: >
      Foundation fund master - the canonical fund entity for downstream consumption.
      One record per fund. Contract-enforced schema. Breaking changes require version increment.
    config:
      materialized: incremental
      schema: foundation
      tags: ['foundation', 'investments', 'fund_master']
      contract:
        enforced: true
    access: public
    latest_version: 1
    versions:
      - v: 1
    meta:
      domain: investments
      owner: investment-data-team
      tier: tier1
      pii_contains: false
      sla: "daily by 06:00 UTC"
    columns:
      - name: fund_id
        data_type: varchar
        description: "Canonical fund identifier - unique, non-null"
        constraints:
          - type: primary_key
        meta:
          pii: false
          source_system: "gp_quarterly_report, admin_fund_data"
        tests:
          - unique
          - not_null
      - name: fund_name
        data_type: varchar
        description: "Fund name as reported by the GP"
        meta:
          source_system: gp_quarterly_report
      - name: legal_name
        data_type: varchar
        description: "Fund legal name from administrator"
        meta:
          source_system: admin_fund_data
      - name: gp_name
        data_type: varchar
        description: "General partner name"
      - name: strategy
        data_type: varchar
        description: "Investment strategy"
      - name: strategy_name
        data_type: varchar
        description: "Canonical strategy name from reference"
      - name: asset_class
        data_type: varchar
        description: "Asset class classification"
      - name: strategy_group
        data_type: varchar
        description: "High-level strategy grouping"
      - name: vintage_year
        data_type: integer
        description: "Fund vintage year"
        constraints:
          - type: check
            expression: "vintage_year >= 1990 AND vintage_year <= 2030"
      - name: currency
        data_type: varchar
        description: "Fund base currency (ISO 4217)"
        constraints:
          - type: not_null
      - name: currency_name
        data_type: varchar
        description: "Full currency name"
      - name: domicile
        data_type: varchar
        description: "Fund domicile jurisdiction"
      - name: fund_structure
        data_type: varchar
        description: "Legal structure (LP, LLC, etc.)"
      - name: inception_date
        data_type: date
        description: "Fund inception date"
      - name: administrator_name
        data_type: varchar
        description: "Fund administrator name"
      - name: committed_capital_mm
        data_type: "decimal(18,4)"
        description: "Total committed capital in millions"
        constraints:
          - type: not_null
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
      - name: called_capital_mm
        data_type: "decimal(18,4)"
        description: "Capital called to date in millions"
      - name: nav_mm
        data_type: "decimal(18,4)"
        description: "Net asset value in millions"
      - name: distributed_mm
        data_type: "decimal(18,4)"
        description: "Distributions to date in millions"
      - name: tvpi
        data_type: "decimal(18,4)"
        description: "Total Value to Paid-In ratio"
      - name: dpi
        data_type: "decimal(18,4)"
        description: "Distributions to Paid-In ratio"
      - name: unfunded_commitment_mm
        data_type: "decimal(18,4)"
        description: "Remaining unfunded commitment in millions"
      - name: is_active
        data_type: boolean
        description: "Whether the fund is currently active"
      - name: fund_age_years
        data_type: integer
        description: "Fund age in whole years from inception"
      - name: reporting_quarter
        data_type: varchar
        description: "Most recent reporting period"
      - name: report_date
        data_type: date
        description: "Date of the most recent GP report"
      - name: usd_fx_rate
        data_type: "decimal(18,4)"
        description: "Exchange rate to USD"
      - name: has_data_quality_flag
        data_type: boolean
        description: "Whether this record has a data quality flag"
      - name: data_quality_flag_reason
        data_type: varchar
        description: "Reason for data quality flag (null if no flag)"
      - name: strategy_lookup_missing
        data_type: boolean
        description: "Whether the strategy lookup failed"
```

---

## Additional Files

### Singular Tests

**tests/singular/assert_fund_master_tvpi_non_negative.sql**
```sql
-- Test: TVPI must be non-negative when calculated
-- Severity: error
-- Generated from: foundation.fund_master data_validation (post)

select *
from {{ ref('fnd_fund_master') }}
where tvpi is not null and tvpi < 0
```

**tests/singular/assert_fund_master_called_le_committed.sql**
```sql
-- Test: Called capital must not exceed committed capital
-- Severity: error
-- Generated from: foundation.fund_master data_validation (post)

select *
from {{ ref('fnd_fund_master') }}
where called_capital_mm > committed_capital_mm + 0.01
```

### Quarantine Model

**models/intermediate/int_fund_master_quarantine.sql**
```sql
-- Model: int_fund_master_quarantine
-- Pattern: data_validation (quarantine)
-- Records that failed error-severity validation rules
-- Abort threshold: 1% error rate, 5% warn rate

{{ config(
    materialized='table',
    schema='intermediate',
    tags=['foundation', 'investments', 'fund_master', 'quarantine']
) }}

with source as (

    select * from {{ ref('int_fund_master_curated') }}

),

quarantined as (

    -- Rule: Committed capital must be positive
    select
        *,
        'committed_capital_must_be_positive' as _quarantine_rule,
        'error' as _quarantine_severity,
        current_timestamp as _quarantine_timestamp,
        'committed_capital_mm' as _quarantine_field
    from source
    where not (committed_capital_mm >= 0)

    union all

    -- Rule: TVPI must be non-negative when calculated
    select
        *,
        'tvpi_non_negative' as _quarantine_rule,
        'error' as _quarantine_severity,
        current_timestamp as _quarantine_timestamp,
        'tvpi' as _quarantine_field
    from source
    where tvpi is not null and tvpi < 0

    union all

    -- Rule: Called capital must not exceed committed capital
    select
        *,
        'called_le_committed' as _quarantine_rule,
        'error' as _quarantine_severity,
        current_timestamp as _quarantine_timestamp,
        'called_capital_mm' as _quarantine_field
    from source
    where called_capital_mm > committed_capital_mm + 0.01

)

select * from quarantined
```

---

## Required dbt Packages

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.3.0", "<2.0.0"]
```

---

## Summary

| # | File | Pattern | Lines |
|---|------|---------|-------|
| 1 | models/sources/_sources.yml | batch_transfer | ~100 |
| 2 | models/intermediate/int_fund_master_mapped_gp.sql | schema_transform + calculated_fields | ~75 |
| 3 | models/intermediate/int_fund_master_mapped_admin.sql | schema_transform + calculated_fields | ~50 |
| 4 | models/intermediate/int_fund_master_curated.sql | data_curation + enrichment + filtering | ~110 |
| 5 | models/foundation/fnd_fund_master.sql | final foundation model | ~55 |
| 6 | models/intermediate/_int_fund_master__models.yml | intermediate schema | ~80 |
| 7 | models/foundation/_fnd_fund_master__models.yml | foundation schema + contract | ~130 |
| 8 | tests/singular/assert_fund_master_tvpi_non_negative.sql | post-validation | ~8 |
| 9 | tests/singular/assert_fund_master_called_le_committed.sql | post-validation | ~8 |
| 10 | models/intermediate/int_fund_master_quarantine.sql | quarantine | ~50 |
