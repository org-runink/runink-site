---
title: "Runink Quickstart"
weight: 120
---
# 🚀 Runink Quickstart: CDM Trade Pipeline

This example shows how to define, test, apply, and run a declarative data pipeline using Runink.

---

## Environment scenario

```mermaid
%% Mermaid Diagram: Runink Architecture (Blueprint View)

flowchart TD
  subgraph Developer_Client["🌐 Developer / Client"]
    Developer["Developer"]
  end

  subgraph Global_Control_Plane["🧭 Runink Global Control Plane (HA Setup)"]
    GlobalAPI["API Server x3
- AuthN/AuthZ
- Herd Routing
- TLS gRPC"]
    HerdDirectory["Herd Directory
- Maps Herds to Raft Groups
- Metadata Routing"]
  end

  subgraph Finance_Herd["🏦 Finance Herd Partition"]
    FinanceScheduler["Finance Scheduler (Leader)
- DAG Planning
- Placement Decisions"]
    FinanceBarn["Finance Barn (KV Store)
- BadgerDB (Local)"]
    FinanceGovernance["Finance Governance Service
- Lineage
- Quality
- Contracts"]
    FinanceSecrets["Finance Secrets Manager
- Raft-backed Secret Storage"]
    FinanceRaft["Finance Raft Group (5 Nodes)
(etcd-io/raft)"]
  end

  subgraph Analytics_Herd["📊 Analytics Herd Partition"]
    AnalyticsScheduler["Analytics Scheduler (Leader)
- DAG Planning
- Placement Decisions"]
    AnalyticsBarn["Analytics Barn (KV Store)
- BadgerDB (Local)"]
    AnalyticsGovernance["Analytics Governance Service
- Lineage
- Quality
- Contracts"]
    AnalyticsSecrets["Analytics Secrets Manager
- Raft-backed Secret Storage"]
    AnalyticsRaft["Analytics Raft Group (5 Nodes)
(etcd-io/raft)"]
  end

  subgraph Worker_Cluster["🧱 Worker Nodes Cluster"]
    RuniAgent["Runi Agent x100
- Node Registration
- Slice Management
- Metrics Collection"]
    RuniSlice["Runi Slice (Ephemeral Container)
- Herd Namespaced
- Config Loaded
- Secrets Injected"]
  end

  Developer --> | CLI/API Requests | GlobalAPI 
  GlobalAPI --> | Resolve Herd Assignment | HerdDirectory 
  GlobalAPI --> | Finance Pipelines | FinanceScheduler 
  GlobalAPI --> | Analytics Pipelines | AnalyticsScheduler 

  FinanceScheduler --> | DAG and Placement Reads | FinanceBarn 
  FinanceGovernance --> | Metadata/Lineage Writes | FinanceBarn 
  FinanceSecrets --> | Secrets CRUD | FinanceBarn
  FinanceBarn --> | Log Replication | FinanceRaft

  AnalyticsScheduler --> | DAG and Placement Reads | AnalyticsBarn
  AnalyticsGovernance --> | Metadata/Lineage Writes | AnalyticsBarn
  AnalyticsSecrets --> | Secrets CRUD | AnalyticsBarn 
  AnalyticsBarn --> | Log Replication | AnalyticsRaft

  FinanceScheduler --> | Dispatch Finance Slices | RuniAgent
  AnalyticsScheduler --> | Dispatch Analytics Slices | RuniAgent

  RuniAgent --> | Launch with Herd Isolation | RuniSlice
  RuniAgent --> | Fetch Finance Secrets | FinanceSecrets
  RuniAgent --> | Fetch Analytics Secrets | AnalyticsSecrets

  RuniSlice --> | Emit Lineage Events | FinanceGovernance
  RuniSlice --> | Emit Lineage Events | AnalyticsGovernance 
  RuniSlice --> | Expose Service Port | RuniAgent 
  RuniAgent --> | Port-Forwarded Access | Developer 
```

## 🛠️ Prerequisites

Ensure you have:

- A `.dsl` scenario: `features/cdm_trade/trade_cdm.dsl`
- A Go contract file: `contracts/trade_cdm_multi.go`
- Golden test files: `golden/cdm_trade/`
- Sample input data: `golden/cdm_trade/input.json`

Our example presents the following:

```mermaid
flowchart TD
  A["Kafka (raw)"] --> B["DecodeCDMEvents"]
  B --> C["sf:control.decoded_cdm_events"]

  B --> D["ValidateLifecycle"]
  D -->|if !valid| E["sf:control.invalid_cdm_events"]
  D -->|if valid| F["TagWithFDC3Context"]
  F --> G["sf:cdm.validated_trade_events"]
```

---

## 💡 Example Flow

```bash
# Create a secure namespace (herd)
runi herd init finance
runi compile --scenario features/payment.dsl --herd finance --contract contracts/payment.go --out dags/payment.go
runi run --dag dags/payment.go
```
---

## 🧪 Test Your Pipelines

```bash
runi audit --scenario features/payment.dsl --contract contracts/payment.go --golden tests/input.json
runi synth --scenario features/payment.dsl --contract contracts/payment.go --golden tests/input.json
runi fetch --scenario features/example.dsl --golden tests/input.json --output table.sql --show
```

---

## 📊 Inspect Pipeline Execution

After running, inspect the pipeline using:

```bash
runi status --run-id RUN-20240424-XYZ --herd finance
```

### Example Output

```yaml
run_id: RUN-20240424-XYZ
herd: finance
status: completed
steps:
  - DecodeCDMEvents:
      processed: 2
      output: sf://control.decoded_cdm_events
  - ValidateLifecycle:
      passed: 1
      failed: 1
      output: 
        - valid → sf://cdm.validated_trade_events
        - invalid → sf://control.invalid_cdm_events
  - TagWithFDC3Context:
      enriched: 1
      context_prefix: fdc3.instrumentView:
lineage:
  contract_hash: a9cd23f…
  contract_version: v3
  created_by: service-account:etl-runner
```

---

## 🔍 Follow-Up Commands

```bash
runi lineage --run-id RUN-20240424-XYZ
runi logs --run-id RUN-20240424-XYZ
runi publish --herd finance --scenario features/cdm_trade/cdm_trade.dsl
```

---

Runink makes secure, declarative data orchestration easy — every pipeline is testable, auditable, and reproducible.
