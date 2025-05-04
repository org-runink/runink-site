---
title: "Runink Docs"
weight: 1
---

## 📌 Overview

**Runink** is a Go-native distributed pipeline orchestration and governance platform. It defines a **self-sufficient, distributed environment** to orchestrate and execute data pipelines — replacing complex Kubernetes or Slurm setups with an integrated runtime built on:

- Linux primitives: `cgroups`, `namespaces`, `exec`
- Go-based execution and scheduling
- Governance, lineage, and contract enforcement
- Serverless-style, per-slice isolation and resource control

**Runink slices run like fast, secure micro-VMs — written in Go, isolated with Linux, coordinated by Raft.**

---

## 🔑 Core Principles

- **Go Native & Linux Primitives**: Minimal overhead, high performance
- **Self-Contained Cluster**: No external orchestrator needed
- **Serverless Execution Model**: Declarative pipelines, smart scheduling
- **Security-First**: OIDC, RBAC, secrets, mTLS
- **Data Governance Built-In**: Contracts, lineage, auditability
- **Observability**: Prometheus-ready, structured logs, golden testing

---

## 🚀 Key Features

![Components Diagram](/images/components.png)

- DAG compilation from `.dsl` + contracts
- Runi slice execution (lightweight, isolated)
- Namespace-based Herd isolation
- Secure secrets vault via Raft
- Built-in schema contracts and golden testing
- Metadata lineage and compliance APIs

---

## 🧠 System Concepts


<style>
table, th, td {
  border: 0px;
  border-collapse: collapse;
  background-color: #FFFFFF;
}
</style>
<table>
<caption><b><i>Runink: DAG compiler from Domain Structured Language (DSL) files and related data contracts.</b></i></caption>
  <tr>
    <td><img src="/images/runink.png" width="250"/></td>
    <td>The golang code base to deploy features from configurations files deployed by command actions over the CLI/API.</td>
  </tr>
</table>
<br>
<table>
<caption><b><i>Runi: Lifecycle workload among agents and its available workers, Dag pipeline slice scheduling and execution.</b></i></caption>
  <tr>
    <td><img src="/images/runi.png" height="140" width="320"/></td>
    <td>A single instance of a pipeline step running as an isolated <i>Runi Slice Process</i> managed by a <i>Runi Agent</i> within the constraints of a specific <i>Herd</i></td>
  </tr>
</table>
<br>
<table>
<caption><b><i>Herds: Domain boundaries, Secrets Management, gRPC/Rest Services, Runi control plane.</b></i></caption>
  <tr>
    <td><img src="/images/herd.png" width="350"/></td>
    <td>A logical grouping construct, similar to a Kubernetes Namespace, enforced via RBAC policies and resource quotas. Provides multi-tenancy and domain isolation.</td>
  </tr>
</table>
<br>
<table>
<caption><b><i>Barn: Metadata governance, lineage tracking</b></i></caption>
  <tr>
    <td><img src="/images/barn.png" width="350"/></td>
    <td>A distributed, Raft-backed state store that guarantees strong consistency, high availability, and deterministic orchestration. No split-brain, no guesswork — just fault-tolerant operations.</td>
  </tr> 
</table>
<br>

---

## 🛠 Getting Started

```bash
# Bootstrap your herd
runi herd init my-data-herd

# Compile DSL → DAG
runi compile \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --out dags/trade_cdm_dag.go

# Generate test data
runi synth \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --golden cdm_trade_input.json

# Run validation
runi audit \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --golden cdm_trade_input.json

# Execute pipeline
runi run --dag dags/trade_cdm_dag.go
````

---

## 🧭 Docs & Exploration

* [📘 Architecture](/docs/architecture/)
* [🔎 Benchmark Comparison](/docs/benchmark/)
* [🧱 Component Overview](/docs/components/)
* [📚 Documentation Home](/docs/)
* [💬 Community Forums](/forums/)

---

## 🧪 Project Status

**Alpha / Experimental** — Actively evolving. Feedback-driven development. Architecture represents target vision.

---

## 🤝 Contributing

We welcome PRs, issues, and feedback!
Start with our [contribution guide](/docs/contributing/).

---

## 📜 License

Apache 2.0 — [View LICENSE on GitHub](https://github.com/paesdan/runink/blob/main/LICENSE)