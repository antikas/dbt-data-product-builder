# Data Product Framework

**Created:** 2026-03-07 (rebuilt from Oct–Dec 2024 discussions)
**Status:** Active
**Nature:** Pure logical architecture — platform and technology agnostic
**Related:** [pipeline-building-blocks.md](pipeline-building-blocks.md)

---

## The Foundational Principle: Business Logic Outlives Technology

The sources, targets, business rules, quality thresholds, transformation specifications, and data contracts that define a data product change far less frequently than the technology used to execute them. Business rules evolve on a quarterly or annual cadence. Technology platforms change on a multi-year cadence — and when they do, the change is total.

**This asymmetry is the entire reason the framework exists.** If business logic is captured declaratively in config and contracts — engine-agnostic, vendor-neutral — then a technology migration (legacy warehouse → cloud, BigQuery → Snowflake, Spark → dbt → something else) is a framework concern: write or update a target translator. The business logic, quality rules, and contracts do not change. Migration becomes a one-time engineering effort on the framework, not a per-pipeline recode.

Without this separation, every platform migration is a line-by-line rewrite of the same business logic in new syntax, delivering zero business value at enormous cost. Organisations that have been through this know the pain — 18-month migration projects where thousands of stored procedures are manually translated from Oracle PL/SQL to Spark, or from Informatica mappings to dbt models, doing exactly the same thing as before. The framework eliminates that class of work.

**Implication for framework design:** The YAML config layer must never contain engine-specific syntax, API references, or technology assumptions. Anything engine-specific belongs in the target translator, not the config. If you find yourself writing Spark SQL or dbt Jinja in the YAML, you have violated this principle.

---

## What Is a Data Product?

A data product is a dataset treated as a first-class product — with a defined owner, a schema contract, quality SLAs, documented lineage, and a discoverable interface. It is not a table produced by a pipeline. The distinction matters:

| Pipeline output | Data product |
|----------------|--------------|
| Exists to move data | Exists to serve a consumer need |
| Owned by engineers | Owned by a domain |
| Changed freely | Changed through versioning |
| Quality implied | Quality contracted |
| Discovered by accident | Discoverable by design |

The product metaphor forces three disciplines: ownership, contractual quality, and consumer orientation.

---

## The Layered Architecture

The framework organises data into four logical layers. These map to different names depending on implementation context, but the logical distinctions are consistent:

```
Source Systems
      │
      ▼
┌─────────────────────────────────┐
│  LAYER 1 — Origin / Raw         │  "staging" in dbt, "bronze" in medallion
│  Mirror of source, no logic     │
└─────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────┐
│  LAYER 2a — Foundation (Base)   │  "intermediate" in dbt, "silver" in medallion
│  Entity-centric, conformed,     │
│  canonical golden records       │
└─────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────┐
│  LAYER 2b — Foundation          │  "intermediate/marts" in dbt, upper "silver"
│  (Feature / Consolidated)       │  or domain-specific "gold"
│  Cross-entity, derived,         │
│  ML-ready, aggregated           │
└─────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────┐
│  LAYER 3 — Consumption          │  "marts" in dbt, "gold" in medallion
│  Purpose-built for a specific   │
│  consumer or use case           │
└─────────────────────────────────┘
```

---

## Layer 1 — Origin Data Products

**What they are:** An exact, immutable mirror of what arrived from the source system. No business logic. No transformation beyond format normalisation for storage.

**Core characteristics:**
- Preserve source truth — if the source had a bug, the origin product preserves it
- Insert-only where possible — no updates, no deletes
- Tagged at landing: source system, arrival timestamp, reporting period
- Schema is source-native (not yet conformed to enterprise model)
- Consumer: other data products, not end users

**What they answer:** "What did the source system say, and when did we receive it?"

**The discipline:** Resist the temptation to enrich at this layer. The moment you add derived fields, you introduce assumptions that obscure the ground truth. Debugging data quality issues requires being able to click through to exactly what arrived.

**dbt parallel:** staging models — direct reads from raw sources, minimal transformations (casting, renaming, deduplication), no joins across sources.

**Medallion parallel:** bronze layer — land whatever arrives, never reject, schema enforcement deferred to the next layer.

---

## Layer 2a — Foundation Data Products (Base)

**What they are:** Curated, entity-centric, governed representations of the business. Multiple source origin products are integrated into a single canonical representation of each entity type. Business semantics begin here.

**Core characteristics:**
- One golden record per entity instance — entity resolution happens at this layer
- Conforms to a canonical schema (enterprise data model / common schema)
- Versioned — consumers can depend on a stable interface
- Quality SLAs: completeness thresholds, null rate bounds, schema versioning
- Reusable — not built for one team, owned by the entity domain
- Temporal history is explicit — SCDs, event history, or ValidSince-based open-ended historisation

**What they answer:** "Who is this entity, and what do we factually know about them as of a given point in time?"

**The base FDP distinction:** These are facts about entities. They do not aggregate, derive scores, or compute cross-entity signals. A customer base FDP knows the customer's attributes. It does not know their propensity to churn.

**Entity examples (domain-agnostic):**
- Customer / Counterparty / Entity master
- Account / Position / Instrument
- Transaction (enriched, normalised, conformed)
- Product / Fund / Contract

**dbt parallel:** intermediate models — joins across staging sources, business key resolution, conformance to common model.

**Medallion parallel:** silver layer — conformed, quality-checked, entity-resolved.

---

## Layer 2b — Foundation Data Products (Feature / Consolidated)

**What they are:** Derived representations computed from base FDPs. They cross entity boundaries, apply business logic, compute signals, or prepare data for specific downstream uses such as ML feature serving. They are still governed data products — not ad-hoc analytics.

**Why this distinction matters:** Base FDPs describe entities. Feature/consolidated FDPs interpret relationships between entities and compute derived meaning. The separation keeps base FDPs stable and broadly reusable while allowing feature evolution without destabilising the entity layer.

**Sub-types:**

### Consolidated / Cross-Domain FDPs
Aggregate or join across multiple base entity FDPs to produce cross-domain views.

Examples:
- Customer relationship graph (customer + accounts + products + transactions)
- Portfolio exposure by sector (fund + position + instrument + classification)
- Risk summary by counterparty (entity + transaction + exposure + limit)

### Feature Data Products (for ML)
ML-ready representations computed from base FDPs, transformed into signals that models can consume.

**Offline features (training):** Point-in-time correct snapshots for training dataset assembly. Must preserve what the feature value was at the time of a historical event — not what it is today. This is the point-in-time correctness problem. Look up the Twine algorithm (from Anchor Modeling) for an efficient approach to temporal joins.

**Online features (serving):** Low-latency versions of the same features for real-time inference. Must be computed and stored differently from the offline versions, but using identical logic — training-serving skew is the most common silent failure mode in production ML.

Feature examples:
- Spending velocity over 30 days
- Months since last adverse event
- Inferred life event flag
- Capital call frequency over 12 months (investment context)
- NAV drift from prior quarter (investment context)

**The training-serving skew trap:** Feature values used to train a model must be identical in logic to values served at inference time. Different transformation code paths, different data sources, or different timing semantics cause the model to degrade in production while appearing healthy in evaluation.

**Prevention:** Single feature computation logic, shared as a data product contract, used by both training pipelines and serving infrastructure.

---

## Layer 3 — Consumption Data Products

**What they are:** Purpose-built data products assembled from foundation FDPs (base or feature/consolidated) for specific consumer domains. Business interpretation, aggregation, and optimisation for a specific use case live here.

**Core characteristics:**
- Domain-specific — built for a use case, not an entity
- May have tighter SLAs than foundation products (fresher, pre-aggregated)
- Consumers: analysts, reporting tools, ML models, APIs, AI agents, regulatory systems
- Derive from FDPs, never from origin products directly

**The distinction from foundation:** Foundation products describe and measure. Consumption products interpret and deliver.

Examples (domain-agnostic):
- Risk exposure dashboard dataset (pre-aggregated, time-partitioned)
- Regulatory reporting extract (exact schema for a specific regulation)
- ML model input set (feature vectors assembled for a model)
- Customer 360 view (enriched, joined, application-ready)
- Investment committee briefing dataset (portfolio summary, current + historical)

---

## The Decoupling Argument

The most important architectural property of this framework: **production is decoupled from consumption**.

Without data products, organisations build point-to-point feeds — each source system feeding each consumer directly. The result is:

```
Source A ──► Consumer 1
Source A ──► Consumer 2
Source A ──► Consumer 3
Source B ──► Consumer 1
Source B ──► Consumer 4
...
```

Hundreds of feeds. When Source A changes schema, every feed breaks independently. When a new consumer appears, a new feed is commissioned. The cost of change is O(sources × consumers).

With the layered data product model:

```
Source A ──► Origin A ──► Foundation ──► Consumption ──► Consumer 1
Source B ──► Origin B ──►              ──► Consumption ──► Consumer 2
                                        ──► Consumer 3
```

Foundation products absorb source changes. Consumption products absorb consumer changes. The cost of change is bounded. A new consumer does not require a new feed to source — it reads from the foundation layer.

---

## Data Contracts

Every data product carries a contract that downstream consumers can depend on. Without contracts, assumptions are invisible until something breaks.

**Contract elements:**

| Element | What it specifies |
|---------|------------------|
| **Identity** | Name, version, owner, domain, product type (origin/foundation/consumption) |
| **Schema** | Fields, types, nullability, semantic meaning, PII flags |
| **Freshness SLA** | Guaranteed currency — daily by 06:00, <5 minutes streaming, etc. |
| **Quality guarantees** | Completeness thresholds, null rate bounds, distribution constraints |
| **Versioning policy** | Major/minor versioning, deprecation notice period |
| **Lineage** | Upstream products this derives from; downstream known consumers |
| **Access** | Classification, permitted roles/teams, row-level security rules |
| **Support** | Owner contact, response time SLA |

**The contract drives everything downstream.** A well-specified contract can auto-generate:
- Schema DDL
- Quality validation rules (dbt tests, Great Expectations)
- Catalog entries
- Access policy
- Lineage registration

**YAML contract sketch (platform-agnostic):**
```yaml
product:
  name: "foundation.customer"
  version: "2.1"
  type: "foundation-base"
  owner: "customer-domain"
  domain: "customer"

schema:
  fields:
    - name: customer_id
      type: string
      required: true
      pii: true
      description: "Canonical customer identifier (entity-resolved)"
    - name: customer_type
      type: string
      required: true
      allowed_values: ["retail", "business", "corporate"]
    - name: valid_since
      type: timestamp
      required: true
      description: "When this record version became effective"

quality:
  freshness: "daily by 06:00 UTC"
  completeness:
    - field: customer_id
      threshold: 1.0
    - field: customer_type
      threshold: 1.0
  validity:
    - rule: "customer_id is non-null and non-empty"

sla:
  availability: 99.9%
  response_time_hours: 4

lineage:
  upstream:
    - "origin.crm_customer"
    - "origin.cbs_customer"
  downstream:
    - "consumption.customer_360"
    - "foundation.customer_features"

versioning:
  policy: "semver"
  deprecation_notice_days: 90
```

---

## The Config-Driven Framework

The framework should be implemented as a **shared codebase** where teams configure rather than build. The core engineering burden (ingestion mechanics, transformation scaffolding, quality gate execution, lineage capture, metadata publishing) is solved once and contributed to by all. Teams building data products supply configuration — the business rules and data-specific transformations — not boilerplate.

**The abstract class pattern (Python sketch):**
```python
from abc import ABC, abstractmethod
from typing import Any, Dict

class DataSource(ABC):
    """Every data product starts with a source. Concrete implementations
    provide the connection logic; the framework provides the scaffolding."""
    @abstractmethod
    def read_data(self, config: Dict[str, Any]) -> Any:
        pass

class DataTransformation(ABC):
    """Business logic lives here. Teams implement transformations;
    the framework handles execution, retry, and lineage capture."""
    @abstractmethod
    def transform(self, data: Any, config: Dict[str, Any]) -> Any:
        pass

class DataSink(ABC):
    """Output target. Teams configure destination;
    the framework handles schema enforcement and quality gates."""
    @abstractmethod
    def write_data(self, data: Any, config: Dict[str, Any]) -> None:
        pass

class DataPipeline:
    """The framework orchestrator. Teams provide a config YAML;
    the framework runs the full cycle: ingest → validate → transform
    → quality check → write → capture lineage → publish metadata."""
    def __init__(self, config: Dict[str, Any]):
        self.config = config
        self.source = self._load_source()
        self.transformations = self._load_transformations()
        self.sink = self._load_sink()

    def run(self):
        data = self.source.read_data(self.config['source'])
        data = self._apply_quality_gates(data, stage='pre')
        for transformation in self.transformations:
            data = transformation.transform(data, self.config)
        data = self._apply_quality_gates(data, stage='post')
        self.sink.write_data(data, self.config['sink'])
        self._capture_lineage()
        self._publish_metadata()
```

**Configuration YAML (what a team actually writes):**
```yaml
pipeline:
  name: "customer_foundation_fdp"
  type: "foundation-base"
  version: "2.1"

source:
  type: "origin_data_product"
  products:
    - "origin.crm_customer"
    - "origin.cbs_customer"

transformations:
  - type: "schema_transform"
    mapping: "configs/customer_schema_map.yaml"
  - type: "entity_resolution"
    key_fields: ["customer_id", "national_id"]
    strategy: "deterministic_then_fuzzy"
  - type: "calculated_fields"
    rules: "configs/customer_business_rules.yaml"
  - type: "data_enrichment"
    reference_data: ["ref.customer_segments", "ref.risk_bands"]

quality:
  pre_checks:
    - "source_completeness"
    - "schema_validation"
  post_checks:
    - "entity_uniqueness"
    - "business_rule_compliance"
  error_threshold: 0.001
  on_failure: "quarantine"

sink:
  type: "foundation_data_product"
  product: "foundation.customer"
  mode: "upsert"

contract: "contracts/foundation.customer.yaml"
```

**The inner source model:** The abstract classes, pipeline orchestrator, quality gate framework, lineage capture, and metadata publishing are maintained by a core platform engineering team. Business unit teams contribute: custom connectors for their source systems, domain-specific transformation rules, and business logic implementations. Contributions go through code review; the core team maintains architectural integrity.

---

## Framework Maturity — 4 Stages

The framework and data product estate evolve through four stages. Do not skip stages; each depends on the stability of the previous.

### Stage 1 — Foundation
Establish the raw capability:
- Basic ingestion (origin products for priority sources)
- Storage with schema versioning
- Essential validation at landing
- Manual deployment

**Exit criterion:** Can reliably land and read source data. Have at least one foundation product end-to-end.

### Stage 2 — Operational
Add resilience and observability:
- Error handling and retry logic
- Basic monitoring (SLA tracking, failure alerting)
- Data cataloguing (products are discoverable)
- Quality gates automated (not manual sign-off)
- Basic data contracts defined

**Exit criterion:** Failure is detected automatically. Consumers can find and understand products without asking engineers.

### Stage 3 — Optimised
Automation and self-service:
- Configuration-driven pipeline creation (teams configure, not code)
- End-to-end CI/CD for data products
- Performance tuning and cost management
- Advanced monitoring (drift detection, SLA forecasting)
- Self-service consumption layer

**Exit criterion:** A new data product can be created by a domain team without core platform engineers doing the work.

### Stage 4 — Intelligent
Automation with intelligence:
- ML-based anomaly detection on data quality
- Automated lineage impact analysis
- Predictive SLA management
- Autonomous remediation for known failure patterns
- Data product usage analytics driving prioritisation

**Exit criterion:** The platform surfaces problems before consumers notice them.

---

## Guiding Principles

1. **One product, many consumers** — replace P2P feeds with a product everyone reads
2. **Contracts before code** — define the interface before building the pipeline
3. **Origin products never lie** — preserve source truth exactly; bugs in source = bugs in origin
4. **Base FDPs describe; feature FDPs interpret** — keep the separation clean
5. **Training-serving skew is a data product problem** — single feature definition, served in both contexts
6. **Point-in-time correctness is non-negotiable** — historical facts must reflect what was known at the time
7. **Configuration over code** — teams should supply business rules, not boilerplate
8. **Governance at the product, not the query** — contracts, lineage, and access control live at product level
9. **Business logic outlives technology** — see [The Foundational Principle](#the-foundational-principle-business-logic-outlives-technology) at the top of this document. This is the strategic reason the framework exists.

---

## TODO

- [ ] Expand entity resolution section — deterministic vs probabilistic matching, golden record merge rules
- [ ] Document data contract version management — what constitutes a breaking vs non-breaking change
- [ ] Feature store architecture — offline vs online serving, latency requirements, vendor landscape
- [ ] Data quality operating model — the 4-layer model, controls result store, escalation paths
- [ ] Feature governance — permissibility rules for ML (what features can drive which decisions)
