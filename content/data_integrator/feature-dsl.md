---
title: "Feature DSL"
weight: 80
---

# 📘 Feature DSL — Authoring Pipelines in Natural Language

Runink’s `.dsl` files allow data, governance, and domain teams to write **declarative pipeline logic** in plain English — no YAML, no code, just structured steps tied to contracts.

Inspired by **Gherkin** and **feature-driven development**, the DSL is intentionally designed to:
- Align with real-world **data contracts**
- Support **lineage**, **compliance**, and **multi-tenant governance**
- Be editable by **non-engineers** — analysts, stewards, and reviewers

---

## ✨ Full Example

```feature
Feature: Trade Events – Validation & Compliance

Scenario: Validate and Tag Incoming Financial Trade Events

  Metadata:
    purpose: "Check and tag incoming trade events for compliance and data quality"
    module_layer: "Silver"
    herd: "Finance"
    slo: "99.9%"
    classification: "pii"
    contract: "cdm_trade/fdc3events.contract"
    contract_version: "1.0.0"
    compliance: ["SOX", "GDPR", "PCI-DSS"]
    lineage_tracking: true

  Given: "Arrival of Events"
    - source_type: stream
    - name: "Trade Events Kafka Stream"
    - format: CDM
    - tags: ["live", "trades", "finance"]

  When "Data is received":
    - Decode each trade event using the CDM schema
    - Check for required fields: trade_id, symbol, price, timestamp
    - Mask sensitive values like SSNs, emails, and bank accounts
    - Tag events with classification and region
    - Compare schema to the latest approved contract version

  Then:
    - Send valid records to: snowflake table "Validated Trades Table"
    - Send invalid records to: snowflake table "DLQ for Invalid Trades"
    - Annotate all records with compliance and lineage metadata

  Assertions:
    - At least 1,000 records must be processed
    - Schema drift must not be detected
    - All sensitive fields must pass redaction or tokenization checks

  GoldenTest:
    input: "cdm_trade/fdc3events.input"
    output: "cdm_trade/data/fdc3events.validated.golden"
    validation: strict

  Notifications:
    - On schema failure → alert "alerts/finance_data_validation"
    - On masking failure → alert "alerts/finance_security_breach"
````

---

## 🧠 DSL Concepts

| Block           | Description                                                 |
| --------------- | ----------------------------------------------------------- |
| `Feature`       | High-level business intent (group of scenarios)             |
| `Scenario`      | Specific pipeline run, often tied to a contract version     |
| `Metadata`      | Tags used for governance, lineage, compliance, and SLOs     |
| `Given`         | Declares the data source and input assumptions              |
| `When`          | Describes logic, transformations, and validations to apply  |
| `Then`          | Declares output actions — writing to sinks, tagging, alerts |
| `Assertions`    | Validate record counts, masking, schema drift, etc.         |
| `GoldenTest`    | Points to expected inputs/outputs for regression safety     |
| `Notifications` | Alerts emitted when failures occur during pipeline runs     |

---

## 🔍 Metadata-Driven Pipelines

Each `.dsl` is **contract-aware** and **herd-scoped** by default.

* Contracts are declared via `contract:` and `contract_version:`
* SLOs, classification, and compliance tags are baked into `Metadata`
* Data lineage and observability are auto-inferred from DSL structure

---

## ✅ DSL Advantages

* **Self-documenting**: Reads like an audit policy + data spec
* **Secure-by-default**: Every scenario runs inside a `herd`
* **Governance-friendly**: Tracks lineage, policy, SLOs, classification
* **Reproducible**: `GoldenTest` ensures outputs match expectations
* **Declarative**: No code, no imperative orchestration logic

---

## 📎 Tips for Authors

| Use this                     | Instead of             |
| ---------------------------- | ---------------------- |
| `- Mask sensitive values...` | `@step("FieldMasker")` |
| `"Validate and Tag..."`      | `"run pipeline X"`     |
| Plain-English assertions     | Inline test logic      |
| `contract: "x.contract"`     | Hardcoded field lists  |

---

## 📚 Related Links

* [📑 Schema Contracts](./schema-contracts.md)
* [🧬 Data Lineage](./data-lineage.md)
* [🧪 Testing Pipelines](./getting_started.md#5-test-your-pipelines)

---

## 🎯 Final Thought

With Runink DSL, **your data pipeline is the spec** — no translation layers, no wasted doc effort. Write what the pipeline should do, tag it with intent, and Runink turns it into secure, auditable, executable logic.

Let your domain experts lead the way — and let infra follow automatically.