# Customer Orders Example - Expected Generator Output

**Config:** `foundation.customer` v2.1.0
**Generated artefacts:** 7 files (no quarantine, no singular tests)

---

## File 1: models/sources/_sources.yml

```yaml
version: 2

sources:
  - name: crm_customer
    schema: raw
    description: "Customer records from the CRM system"
    tables:
      - name: crm_customers
        description: "Customer records from the CRM system"
        columns:
          - name: customer_id
            description: "CRM-assigned customer identifier"
            tests:
              - not_null
              - unique
          - name: email
            description: "Primary email address"
            tests:
              - dbt_expectations.expect_column_values_to_match_regex:
                  regex: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
                  severity: warn
          - name: customer_type
            description: "Customer classification (retail, business, corporate)"
            tests:
              - accepted_values:
                  values: ["retail", "business", "corporate"]

  - name: billing_customer
    schema: raw
    description: "Customer records from the billing/invoicing system"
    tables:
      - name: billing_customers
        description: "Customer records from the billing/invoicing system"
        columns:
          - name: customer_id
            description: "Billing-assigned customer identifier (matches CRM)"
            tests:
              - not_null
              - unique
```

---

## File 2: models/intermediate/int_customer_mapped_crm.sql

```sql
-- Model: int_customer_mapped_crm
-- Pattern: schema_transform + calculated_fields
-- Source: crm_customer
-- Generated from: foundation.customer v2.1.0

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'customer']
) }}

with source as (

    select * from {{ source('crm_customer', 'crm_customers') }}

),

renamed as (

    select
        cast(customer_id as varchar) as customer_id,
        cast(first_name as varchar) as first_name,
        cast(last_name as varchar) as last_name,
        cast(email as varchar) as email,
        cast(phone as varchar) as phone,
        cast(customer_type as varchar) as customer_type,
        cast(country_code as varchar) as country_code,
        cast(status_code as varchar) as status_code,
        cast(created_date as date) as created_date,
        cast(last_activity_date as date) as last_activity_date
    from source

),

-- Calculated fields (rule version: 2026Q1, owner: customer-domain)
calculated as (

    select
        *,
        -- Full customer name (first + last)
        trim(first_name || ' ' || last_name) as full_name,
        -- Whether the customer is currently active
        case when status_code in ('A', 'P') and last_activity_date >= current_date - interval '365 days' then true else false end as is_active,
        -- Customer tenure in months since account creation
        date_part('month', age(current_date, created_date)) + date_part('year', age(current_date, created_date)) * 12 as customer_tenure_months
    from renamed

)

select * from calculated
```

---

## File 3: models/intermediate/int_customer_mapped_billing.sql

```sql
-- Model: int_customer_mapped_billing
-- Pattern: schema_transform
-- Source: billing_customer
-- Generated from: foundation.customer v2.1.0

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'customer']
) }}

with source as (

    select * from {{ source('billing_customer', 'billing_customers') }}

),

renamed as (

    select
        cast(customer_id as varchar) as customer_id,
        cast(billing_name as varchar) as billing_name,
        cast(billing_email as varchar) as billing_email,
        cast(payment_method as varchar) as payment_method,
        cast(credit_limit as decimal(12,2)) as credit_limit,
        cast(outstanding_balance as decimal(12,2)) as outstanding_balance,
        cast(last_payment_date as date) as last_payment_date,
        cast(account_tier as varchar) as account_tier
    from source

)

select * from renamed
```

---

## File 4: models/intermediate/int_customer_curated.sql

```sql
-- Model: int_customer_curated
-- Pattern: data_curation + enrichment + filtering
-- Entity key: customer_id
-- Resolution: deterministic
-- Sources: 1. int_customer_mapped_crm, 2. int_customer_mapped_billing

{{ config(
    materialized='view',
    schema='intermediate',
    tags=['foundation', 'customer']
) }}

with crm as (

    select * from {{ ref('int_customer_mapped_crm') }}

),

billing as (

    select * from {{ ref('int_customer_mapped_billing') }}

),

-- Entity resolution + survivorship
curated as (

    select
        coalesce(crm.customer_id, billing.customer_id) as customer_id,

        -- CRM preferred (default survivorship)
        crm.full_name,
        crm.first_name,
        crm.last_name,
        crm.email,
        crm.phone,
        crm.customer_type,
        crm.country_code,
        crm.status_code,
        crm.is_active,
        crm.customer_tenure_months,
        crm.created_date,
        crm.last_activity_date,

        -- Billing preferred (field overrides)
        billing.billing_name,
        billing.billing_email,
        billing.payment_method,
        billing.credit_limit,
        billing.outstanding_balance,
        billing.last_payment_date,
        billing.account_tier

    from crm
    full outer join billing on crm.customer_id = billing.customer_id

),

-- Enrichment: customer segment
enriched_segment as (

    select
        curated.*,
        coalesce(ref_seg.segment_name, 'Unclassified') as customer_segment,
        coalesce(ref_seg.service_tier, 'Standard') as service_tier
    from curated
    left join {{ ref('dim_customer_segments') }} as ref_seg
        on curated.customer_type = ref_seg.type_code

),

-- Enrichment: region
enriched_region as (

    select
        enriched_segment.*,
        ref_region.country_name,
        ref_region.region
    from enriched_segment
    left join {{ ref('dim_regions') }} as ref_region
        on enriched_segment.country_code = ref_region.country_code

),

-- Filtering
filtered as (

    select *
    from enriched_region
    where true
        -- Filter: exclude_test_accounts - Exclude internal test and example accounts
        and email not like '%@test.%' and email not like '%@example.%'
        -- Filter: exclude_closed_accounts - Exclude permanently closed accounts
        and status_code != 'C'

)

select * from filtered
```

---

## File 5: models/foundation/fnd_customer.sql

```sql
-- Model: fnd_customer
-- Foundation Data Product: foundation.customer v2.1.0
-- Owner: customer-domain
-- Domain: customer
--
-- Build: dbt build --select fnd_customer+
-- Test:  dbt test --select fnd_customer
-- Docs:  dbt docs generate && dbt docs serve

{{ config(
    materialized='table',
    schema='foundation',
    tags=['foundation', 'customer'],
    contract={'enforced': true}
) }}

with source as (

    select * from {{ ref('int_customer_curated') }}

),

final as (

    select
        customer_id,
        full_name,
        first_name,
        last_name,
        email,
        phone,
        customer_type,
        customer_segment,
        service_tier,
        country_code,
        country_name,
        region,
        status_code,
        is_active,
        customer_tenure_months,
        created_date,
        last_activity_date,
        billing_name,
        billing_email,
        payment_method,
        credit_limit,
        outstanding_balance,
        last_payment_date,
        account_tier
    from source

)

select * from final
```

---

## File 6: models/intermediate/_int_customer__models.yml

```yaml
version: 2

models:
  - name: int_customer_mapped_crm
    description: "Schema transform: CRM customers → canonical customer model with calculated fields."
    config:
      materialized: view
      schema: intermediate
    columns:
      - name: customer_id
        description: "Canonical customer identifier"
        tests:
          - not_null

  - name: int_customer_mapped_billing
    description: "Schema transform: billing customers → canonical customer model."
    config:
      materialized: view
      schema: intermediate
    columns:
      - name: customer_id
        description: "Canonical customer identifier"
        tests:
          - not_null

  - name: int_customer_curated
    description: "Entity resolution: golden record per customer from CRM + billing. CRM preferred for contact, billing for financial. Enriched with segment and region. Test and closed accounts excluded."
    config:
      materialized: view
      schema: intermediate
    columns:
      - name: customer_id
        description: "Canonical customer identifier (entity-resolved)"
        tests:
          - not_null
          - unique
```

---

## File 7: models/foundation/_fnd_customer__models.yml

```yaml
version: 2

models:
  - name: fnd_customer
    description: "Foundation customer entity - one record per customer, contract-enforced."
    config:
      materialized: table
      schema: foundation
      tags: ['foundation', 'customer']
      contract:
        enforced: true
    access: protected
    meta:
      domain: customer
      owner: customer-domain
      tier: tier1
      pii_contains: true
    columns:
      - name: customer_id
        data_type: varchar
        description: "Canonical customer identifier"
        constraints:
          - type: primary_key
        meta:
          pii: true
        tests:
          - unique
          - not_null
      - name: full_name
        data_type: varchar
        description: "Full customer name"
        meta:
          pii: true
      - name: first_name
        data_type: varchar
        description: "Customer first name"
        meta:
          pii: true
      - name: last_name
        data_type: varchar
        description: "Customer last name"
        meta:
          pii: true
      - name: email
        data_type: varchar
        description: "Primary email address"
        meta:
          pii: true
          sensitivity: confidential
      - name: phone
        data_type: varchar
        description: "Primary phone number"
        meta:
          pii: true
          sensitivity: confidential
      - name: customer_type
        data_type: varchar
        description: "Customer classification"
        constraints:
          - type: not_null
          - type: accepted_values
            values: ["retail", "business", "corporate"]
      - name: customer_segment
        data_type: varchar
        description: "Segment classification from reference"
      - name: service_tier
        data_type: varchar
        description: "Service tier"
      - name: country_code
        data_type: varchar
        description: "ISO country code"
      - name: country_name
        data_type: varchar
        description: "Full country name"
      - name: region
        data_type: varchar
        description: "Geographic region"
      - name: status_code
        data_type: varchar
        description: "Account status code"
      - name: is_active
        data_type: boolean
        description: "Whether the customer is currently active"
      - name: customer_tenure_months
        data_type: integer
        description: "Tenure in months"
      - name: created_date
        data_type: date
        description: "Account creation date"
      - name: last_activity_date
        data_type: date
        description: "Most recent activity date"
      - name: billing_name
        data_type: varchar
        description: "Name on billing records"
      - name: billing_email
        data_type: varchar
        description: "Billing contact email"
        meta:
          pii: true
      - name: payment_method
        data_type: varchar
        description: "Primary payment method"
      - name: credit_limit
        data_type: "decimal(12,2)"
        description: "Assigned credit limit"
      - name: outstanding_balance
        data_type: "decimal(12,2)"
        description: "Current outstanding balance"
      - name: last_payment_date
        data_type: date
        description: "Most recent payment date"
      - name: account_tier
        data_type: varchar
        description: "Billing tier"
```

---

## Required dbt Packages

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.3.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<1.0.0"]
```

Note: `dbt_expectations` is required for the email regex validation in pre-validation.

---

## Summary

| # | File | Pattern | Notes |
|---|------|---------|-------|
| 1 | _sources.yml | batch_transfer | 2 sources |
| 2 | int_customer_mapped_crm.sql | schema_transform + calculated_fields | CRM source |
| 3 | int_customer_mapped_billing.sql | schema_transform | Billing source |
| 4 | int_customer_curated.sql | curation + enrichment + filtering | Golden record |
| 5 | fnd_customer.sql | foundation model | Table, contract enforced |
| 6 | _int_customer__models.yml | intermediate schema | 3 models |
| 7 | _fnd_customer__models.yml | foundation schema + contract | PII-tagged |

Compared to the investment example, this config is simpler:
- No deduplication (sources are clean)
- No quarantine (simpler DQ approach)
- No incremental (full refresh)
- No singular tests (all validation maps to schema-level tests)
- PII tagging demonstrates governance metadata
