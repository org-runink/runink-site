---

title: "Runink Docs"
weight: 1
---------

## 📌 Overview

**Runink** is a Go-native distributed pipeline orchestration and governance platform. It defines a **self-sufficient, distributed environment** to orchestrate and execute data pipelines — replacing complex Kubernetes or Slurm setups with an integrated runtime built on:

* Linux primitives: `cgroups`, `namespaces`, `exec`
* Go-based execution and scheduling
* Governance, lineage, and contract enforcement
* Serverless-style, per-slice isolation and resource control

Runink organizes pipelines within **Herds** (namespaces), executes isolated pipeline steps as **Slices**, and centrally manages metadata via a distributed metadata store (**Barn**).

**Runink slices run like fast, secure micro-VMs — written in Go, isolated with Linux, coordinated by Raft.**

---

## 🔑 Core Principles

* **Go Native & Linux Primitives**: Minimal overhead, high performance
* **Self-Contained Cluster**: No external orchestrator needed
* **Serverless Execution Model**: Declarative pipelines, smart scheduling
* **Security-First**: OIDC, RBAC, secrets, mTLS
* **Data Governance Built-In**: Contracts, lineage, auditability
* **Observability**: Prometheus-ready, structured logs, golden testing

---

## 🚀 Key Features

![Runink Components Diagram - illustrates key architecture components including Herds, Slices, and Barn](/images/components.png)

* DAG compilation from `.dsl` + contracts
* Runi slice execution (lightweight, isolated)
* Namespace-based Herd isolation
* Secure secrets vault via Raft
* Built-in schema contracts and golden testing
* Metadata lineage and compliance APIs

---

## 🧠 System Concepts

### 📖 Glossary

* **Herd**: Logical grouping, similar to Kubernetes namespaces, enabling multi-tenancy and isolation.
* **Slice**: An isolated pipeline step process managed securely within a Herd.
* **Barn**: Centralized metadata governance and lineage tracking using Raft for consistency and high availability.

| Component                             | Description                                                                                           |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| ![Runink Diagram](/images/runink.png) | Compiles DSL and data contracts into executable DAGs.                                                 |
| ![Runi Diagram](/images/runi.png)     | Manages slice lifecycle, scheduling, and execution within pipeline DAGs.                              |
| ![Herd Diagram](/images/herd.png)     | Provides domain boundaries, secrets management, and secure isolation.                                 |
| ![Barn Diagram](/images/barn.png)     | Ensures strong consistency, high availability, and deterministic orchestration via Raft-backed store. |

---

## 🛠 Getting Started

Quickly initialize your environment and run your first pipeline:

```bash
# Initialize a new isolated pipeline environment ("herd")
runi herd init my-data-herd

# Compile your pipeline scenario into executable DAG
runi compile \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --out dags/trade_cdm_dag.go

# Generate test input data ("golden files") for your pipeline
runi synth \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --golden cdm_trade_input.json

# Audit your scenario and contracts using generated data
runi audit \
  --scenario features/trade_cdm.dsl \
  --contract contracts/trade_cdm_multi.go \
  --golden cdm_trade_input.json

# Execute the compiled DAG pipeline
runi run --dag dags/trade_cdm_dag.go
```

---

## 🧭 Docs & Exploration

* [📘 Architecture](/docs/architecture/)
* [🔎 Benchmark Comparison](/docs/benchmark/)
* [🧱 Component Overview](/docs/components/)
* [📚 Documentation Home](/docs/)

---

## 🧪 Project Status

**Alpha / Experimental** — Actively evolving. Feedback-driven development. Architecture represents target vision.

---

## 🤝 Contributing

We welcome PRs, issues, and feedback! Start with our [contribution guide](/docs/contributing/).

---

## 📜 License

Apache 2.0 — [View LICENSE on GitHub](https://github.com/paesdan/runink/blob/main/LICENSE)
