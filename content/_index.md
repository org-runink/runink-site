---

title: "Runink: Declarative, Secure Data Pipelines"
weight: 1
layout: "landing"
description: "Runink: Your Go-to Hub for for orchestrating secure, testable, and governance-driven data pipelines at scale. Fitting your Cloud, Data Engineering, and Generative AI initiatives with secure solutions, and cutting-edge compliant technologies."
--------------------------------------------------------------------------------------------------------------------------------

<p align="center">
  <img src="/images/logo.png" alt="Runink Logo" width="220" />
</p>

**Runink** is a self-contained, distributed data orchestration environment — purpose-built to run **secure, declarative data pipelines** without unnecessary extra components.

Written in **Go**, hardened by **Linux namespaces and cgroups**, and coordinated with **Raft**, Runink:

* Uses **Go slices** instead of containers
* Wraps execution with **kernel primitives** (namespaces, cgroups, pipes)
* Orchestrates them with **Raft-consistent DAGs**
* Tracks everything via native **Governance/Observability layers**

Runink is ideal for teams that need:

* 🔐 **Governance** (lineage, contracts, access control)
* ⚡ **Performance** (compiled pipelines, minimal runtime)
* 🧪 **Testing** (golden files, scenario coverage)
* 🧱 **Isolation** (multi-tenant slices within herds)

---

## 🧠 Who is it for?

| For Engineers               | For Organizations                     |
| --------------------------- | ------------------------------------- |
| ✅ Run DAGs with `runi` CLI  | ✅ Enforce data contracts across teams |
| ✅ Version features with Git | ✅ Prove lineage, compliance & SLOs    |

---

## 🧰 How it Works

Each pipeline is defined with:

* **DSL Scenario Files** → declarative `@step`, `@source`, `@sink` specs
* **Contracts** → Go structs with schema and validation logic
* **Herds** → Resource-scoped namespaces for pipeline execution
* **Slices** → Isolated processes launched under cgroups & namespaces

Runink compiles DSL + contracts into DAGs, then schedules them on bare metal, cloud VMs, or edge workers — no orchestrator required.

---

## 🔐 Built for Data Governance

Runink includes:

* ✅ **RBAC & OIDC Integration**
* ✅ **Schema Contracts with Versioning**
* ✅ **Lineage Tracking by Run, Herd, Contract**
* ✅ **Golden Test Framework for Pipeline Scenarios**
* ✅ **Secure Secrets Delivery via Raft-backed store**

---

## 🖼 Architecture Diagram

<p align="center">
  <img src="/images/components.png" alt="Runink Architecture" width="700"/>
</p>

---

## 🛠 Get Involved

* 📚 [Documentation](/docs/)
* 🤝 [Contributing](/docs/contributing/)
* 🧠 [Architecture Overview](/docs/architecture/)
* 🔍 [Benchmark Comparison](/docs/benchmark/)

---

## ⚠️ Development Status

Runink is currently in **alpha** and under active development. Feedback and contributions are welcome.