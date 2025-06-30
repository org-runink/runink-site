---
title: "Schema Contracts"
weight: 130
---
# Schema & Contract Management – Runink

Runink enables **data contracts** as native Go structs — giving you strong typing, version tracking, schema validation, and backward compatibility across pipelines.

This guide shows how to define, version, test, and enforce schema contracts in your pipelines.

---

## 📦 What Is a Contract?
A contract in Runink is a schema definition used to:
- Validate incoming and outgoing data
- Detect schema drift
- Provide PII and RBAC tagging
- Drive pipeline generation and testing

Contracts are generated from Go structs annotated with tags.

---

## ✍️ Defining a Contract

This schema contract defines the structure and policy metadata for incoming FDC3-based trade events. It ensures compliance with financial regulations and enables secure, testable transformations.

```go
package contracts

// ContractName: fdc3events
// Version: 1.0.0
// Classification: pii
// Compliance: SOX, GDPR, PCI-DSS
// AccessPolicy: herd-isolated
// SLO: 99.9%
// Source: Kafka Stream ("topics.trade_events")

type FDC3Event struct {
    TradeID   string  `json:"trade_id" validate:"required" pii:"false"`
    Symbol    string  `json:"symbol" validate:"required" pii:"false"`
    Price     float64 `json:"price" validate:"required" pii:"false"`
    Timestamp string  `json:"timestamp" validate:"required" pii:"false"`

    // PII Fields (Must be masked and tested)
    SSN         string `json:"ssn,omitempty" pii:"true" access:"compliance"`
    BankAccount string `json:"bank_account,omitempty" pii:"true" access:"finance"`
    Email       string `json:"email,omitempty" pii:"true" access:"support"`

    // Metadata for governance tracking
    Region      string   `json:"region,omitempty" lineage:"true"`
    Compliance  []string `json:"compliance_tags,omitempty" lineage:"true"`
    Valid       bool     `json:"valid,omitempty"`
}
````

### 🔐 Field-Level Tags

| Tag                   | Description                                           |
| --------------------- | ----------------------------------------------------- |
| `validate:"required"` | Required for schema validation                        |
| `pii:"true"`          | Field contains sensitive personal data                |
| `access:"role"`       | RBAC-enforced visibility (e.g., `support`, `finance`) |
| `lineage:"true"`      | Field tracked for lineage and audit logging           |

---

## 🧪 Decode Stage

This function is the first `@step` in most `.dsl` scenarios using this contract:

```go
// DecodeFDC3Events parses raw CDM Kafka events into structured FDC3Event objects.
func DecodeFDC3Events(r io.Reader, w io.Writer) error {
    decoder := json.NewDecoder(r)
    encoder := json.NewEncoder(w)

    for decoder.More() {
        var e FDC3Event
        if err := decoder.Decode(&e); err != nil {
            return err
        }
        encoder.Encode(e)
    }
    return nil
}
```

---

## ✅ Used In:

* `features/fdc3_validation.dsl`
* `golden/cdm_trade/fdc3events.validated.golden.json`
* `fdc3events.conf` as the runtime binding

---


Then run:
```bash
runi contract gen --struct contracts.Order --out contracts/order.json
```

---

## ✅ Enforcing a Contract
```gherkin
Given the contract: contracts/order.json
```
Or:
```bash
runi run --verify-contract
```
Runink ensures that all records match the expected schema.

---

## 🔍 Schema Drift Detection
Compare current vs expected schema:
```bash
runi contract diff --old v1.json --new v2.json
```
Output shows added, removed, or changed fields, types, tags, and ordering.

---

## 📊 Hashing and Snapshotting
Each contract has a hash for:
- Version tracking
- Lineage graph integrity
- Change detection

```bash
runi contract hash contracts/order.json
```

Snapshot for reproducibility:
```bash
runi snapshot --contract contracts/order.json --out snapshots/order_v1.json
```

---

## 🧬 Advanced Tags
- `pii:"true"` – marks field as sensitive
- `access:"finance"` – restricts field to roles
- `enum:"pending,approved,rejected"` – enum constraint (optional)
- `required:"true"` – fail if field is null or missing

---

## 🧪 Contract Testing
Use golden tests to assert schema correctness:
```bash
runi test --scenario features/orders.dsl
```
And diff output against expected:
```bash
runi diff --gold testdata/orders.golden.json --new out/orders.json
```

---

## 🗃️ Contract Catalog
Generate an index of all contracts in your repo:
```bash
runi contract catalog --out docs/contracts.json
```
This can be plugged into:
- Docs browser
- Contract registry
- CI schema check

---

## 🧾 Example Contract Output
```json
{
  "name": "Order",
  "fields": [
    { "name": "order_id", "type": "string" },
    { "name": "amount", "type": "float64" },
    { "name": "notes", "type": "string", "pii": true, "access": "support" }
  ],
  "hash": "a94f3bc..."
}
```

---

## Summary
Contracts in Runink power everything:
- Schema validation
- RBAC and compliance
- Pipeline generation
- Test automation
- Lineage and snapshots

Use contracts to make your data:
- Safe
- Trustworthy
- Documented
- Governed