---
title: "Benchmark"
weight: 20
---


## 1. Architecture & Paradigm

- **Runink:** A Go/Linux-native, vertically integrated data platform that combines execution, scheduling, governance, and observability in a single runtime. Unlike traditional stacks, Runink does not rely on Kubernetes or external orchestrators. Instead, it uses a **Raft-based control plane** to ensure high availability and consensus across services like scheduling, metadata, and security — forming a distributed operating model purpose-built for data.

- **Competitors:** Use a layered, loosely coupled stack:
  - *Execution:* Spark, Beam (JVM-based)
  - *Orchestration:* Airflow, Dagster (often on Kubernetes)
  - *Transformation:* DBT (runs SQL on external data platforms)
  - *Cluster Management:* Kubernetes, Slurm
  - *Governance:* Collibra, Apache Atlas (external)

- **Key Differentiator:** Runink is **built from the ground up** as a distributed system — with **Raft consensus** at its core — whereas competitors compose multiple tools that communicate asynchronously or rely on external state systems.

---

## 1. MapReduce vs. RDD vs. Raft for Data Pipelines

---

### 1.1 Architecture & Paradigm

| Aspect | MapReduce | RDD | Raft (Runink model) |
|:---|:---|:---|:---|
| **Origin** | Google (2004) | Spark (2010) | Raft (2013) adapted for distributed control |
| **Execution Model** | Batch, two-stage (Map → Reduce) | In-memory DAGs of transformations | Real-time coordination of distributed nodes |
| **Consistency Model** | Eventual (job outputs persisted) | Best effort (job outputs in memory, lineage for recovery) | Strong consistency (N/2+1 consensus) |
| **Primary Use** | Large batch analytics | Interactive, iterative analytics | Distributed metadata/state management for pipelines |
| **Fault Tolerance** | Output checkpointing | Lineage-based recomputation | Log replication and state machine replication |

---

### 1.2. Performance & Efficiency

| Aspect | MapReduce | RDD | Raft (Runink) |
|:---|:---|:---|:---|
| **Cold Start Time** | High (JVM startup, slot allocation) | Medium (Spark cluster overhead) | Low (Go processes, native scheduling) |
| **Memory Use** | Disk-heavy | Memory-heavy (RDD caching) | Lightweight (control metadata, not bulk data) |
| **I/O Overhead** | Heavy disk I/O (HDFS reads/writes) | Network/memory optimized, but needs enough RAM | Minimal (only metadata replication) |
| **Pipeline Complexity** | Requires multiple jobs for DAGs | Natural DAG execution | Direct DAG compilation from DSLs (Runink) |

---

### 1.3. Data Governance and Lineage

| Aspect | MapReduce | RDD | Raft (Runink) |
|:---|:---|:---|:---|
| **Built-in Lineage** | No (external) | Yes (RDD lineage graph) | Yes (atomic commit of contracts, steps, runs) |
| **Governance APIs** | Manual (logs, job output) | Partial (Spark listeners) | Native (contracts, lineage logs, per-slice metadata) |
| **Auditability** | Hard to reconstruct | Possible with effort | Native per-run audit logs, Raft-signed events |

---

### 1.4. Fault Tolerance and Recovery

| Aspect | MapReduce | RDD | Raft (Runink) |
|:---|:---|:---|:---|
| **Recovery Mechanism** | Re-run failed jobs | Recompute from lineage | Replay committed log entries |
| **Failure Impact** | Full-stage re-execution | Depends on lost partitions | Minimal if quorum is maintained |
| **Availability Guarantee** | None | Partial (driver failure = job loss) | Strong (as long as majority nodes are alive) |

---

### 1.5. Security and Isolation

| Aspect | MapReduce | RDD | Raft (Runink) |
|:---|:---|:---|:---|
| **Authentication** | Optional | Optional | Mandatory (OIDC, RBAC) |
| **Secrets Management** | Ad hoc | Ad hoc | Native, Raft-backed, scoped by Herds |
| **Multi-Tenancy** | None | None | Herd isolation (namespace + cgroup enforcement) |

---

### 1.6 Real case scenario example

**Imagine a critical pipeline for trade settlement**:

- **MapReduce** would force every job to write to disk between stages — slow and painful for debugging.
- **RDD** would speed things up but require heavy RAM and still risk full job loss if the driver fails.
- **Raft (Runink)** keeps every contract, every transformation, every secret **atomically committed** and recoverable — **even if a node crashes mid-run**, the system can **resume from the last committed stage** safely.

---

## 2. Raft Advantages for Distributed Coordination

**Runink:**  
- Uses **Raft** for **strong consistency** and **leader election** across:
  - Control plane state (Herds, pipelines, RBAC, quotas)
  - Scheduler decisions
  - Secrets and metadata governance

- Guarantees:
  - No split-brain conditions
  - Predictable and deterministic behavior in failure scenarios
  - Fault-tolerant HA (N/2+1 consensus)

**Competitors:**
- Kubernetes uses etcd (Raft-backed), but tools like Airflow/Spark have no equivalent.
- Scheduling decisions, lineage, and metadata handling are often **eventually consistent** or stored in **external systems** without consensus guarantees.
- Result: higher complexity, latency, and coordination failure risks under scale or failure.

---

## 3. Performance & Resource Efficiency

- **Runink:**
  - Written in Go for low-latency cold starts and efficient concurrency.
  - Uses direct `exec`, `cgroups`, and `namespaces`, not Docker/K8s layers.
  - Raft ensures **low-overhead coordination**, avoiding polling retries and state divergence.

- **Competitors:**
  - Spark is JVM-based; powerful but resource-heavy.
  - K8s introduces orchestration latency, plus pod startup and scheduling delays.
  - Airflow relies on Celery/K8s executors with less efficient scheduling granularity.

---

## 4. Scheduling & Resource Management

- **Runink:**
  - Custom, Raft-backed **Scheduler** matches pipeline steps to nodes in real time.
  - Considers Herd quotas, CPU/GPU/Memory availability.
  - Deterministic task placement and retry logic are **logged and replayable** via Raft.

- **Competitors:**
  - Kubernetes schedulers are general-purpose and not pipeline-aware.
  - Airflow does not control actual compute — delegates to backends like K8s.
  - Slurm excels in HPC, but lacks pipeline-native orchestration and data governance.

---

## 5. Security Model

- **Runink:**
  - Secure-by-default with OIDC + JWT, RBAC, Secrets Manager, mTLS, and field-level masking.
  - **Secrets are versioned and replicated with Raft**, avoiding plaintext spillage or inconsistent states.
  - Namespace isolation per Herd.

- **Competitors:**
  - Kubernetes offers RBAC and secrets, but complexity leads to misconfigurations.
  - Airflow often shares sensitive configs (connections, variables) across DAGs.

---

## 6. Data Governance, Lineage & Metadata

- **Runink:**
  - Built-in **Data Governance Service** stores contracts, lineage, quality metrics, and annotations.
  - Changes are **committed to Raft**, ensuring **atomic updates** and rollback support.
  - Contracts and pipeline steps are **versioned** and tracked centrally.

- **Competitors:**
  - Require integrating platforms like Atlas or Collibra.
  - Lineage capture is manual or partial, with data loss possible on failure or drift.
  - Metadata syncing lacks consistency guarantees.

---

## 7. Multi-Tenancy

- **Runink:**
  - Uses **Herds** as isolation units — enforced via RBAC, ephemeral UIDs, cgroups, and namespace boundaries.
  - Raft ensures configuration updates (quotas, roles) are **safely committed** across all replicas.

- **Competitors:**
  - Kubernetes uses namespaces and resource quotas.
  - Airflow has no robust multi-tenancy — teams often need separate deployments.

---

## 8. LLM Integration & Metadata Handling

- **Runink:**
  - LLM inference is a **first-class pipeline step**.
  - Annotations are tied to lineage and **stored transactionally** in the Raft-backed governance store.

- **Competitors:**
  - LLMs are orchestrated as container steps via KubernetesPodOperator or Argo.
  - Metadata is stored in external tools or left untracked.

---

## 9. Observability

- **Runink:**
  - Built-in metrics via Prometheus, structured logs via Fluentd.
  - Metadata and run stats are **Raft-consistent**, enabling reproducible audit trails.
  - Observability spans from node → slice → Herd → run.

- **Competitors:**
  - Spark, Airflow, and K8s use external stacks (Loki, Grafana, EFK) that need configuration and instrumentation.
  - Logs may be disjointed or context-lacking.

---

## 10. Ecosystem & Maturity

- **Runink:**  
  - Early-stage, but intentionally narrow in scope and highly integrated.
  - No need for external orchestrators or data governance platforms.

- **Competitors:**
  - Vast ecosystems (Airflow, Spark, DBT, K8s) with huge community support.
  - Tradeoff: Requires significant integration, coordination, and DevOps effort.

---

## 11. Complexity & Operational Effort

- **Runink:**
  - High initial build complexity — but centralization of Raft and Go-based primitives allows for **deterministic ops**, easier debug, and stronger safety guarantees.
  - **Zero external dependencies** once deployed.

- **Competitors:**
  - Operationally fragmented. DevOps teams must manage multiple platforms (e.g., K8s, Helm, Spark, Airflow).
  - Requires cross-tool observability, secrets management, and governance.

---

## ✅ Summary: Why Raft Makes Runink Different

| Capability | Runink (Raft-Powered) | Spark / Airflow / K8s Stack |
|------------|------------------------|------------------------------|
| **State Coordination** | Raft Consensus | Partial (only K8s/etcd) |
| **Fault Tolerance** | HA Replication | Tool-dependent |
| **Scheduler** | Raft-backed, deterministic | Varies per layer |
| **Governance** | Native, consistent, queryable | External |
| **Secrets** | Encrypted + Raft-consistent | K8s or env vars |
| **Lineage** | Immutable + auto-tracked | External integrations |
| **Multitenancy** | Herds + namespace isolation | Namespaces (K8s) |
| **Security** | End-to-end mTLS + RBAC + UIDs | Complex setup |
| **LLM-native** | First-class integration | Ad hoc orchestration |
| **Observability** | Built-in, unified stack | Custom integration |

## Process flow
{{< mermaid >}}
%%{init:{"fontFamily":"monospace", "sequence":{"showSequenceNumbers":true}}}%%
%% Mermaid Diagram: MapReduce vs RDD vs Raft (Runink)
flowchart LR
  subgraph Normal_Operation["✅ Normal Operation (Execution Flow)"]
    subgraph MapReduce
      ID1["Input Data 📂"]
      MP["Map Phase 🛠️"]
      SS["Shuffle & Sort Phase 🔀"]
      RP["Reduce Phase 🛠️"]
      ID1 --> MP --> SS --> RP
    end

    subgraph RDD
      RID1["Input Data 📂"]
      RT["RDD Transformations 🔄"]
      AT["Action Trigger ▶️"]
      JS["Job Scheduler 📋"]
      CE["Cluster Execution (Executors) ⚙️"]
      OD["Output Data 📦"]
      RID1 --> RT --> AT --> JS --> CE --> OD
    end

    subgraph Raft_Runink
      RIN["Input Data 📂"]
      RC["Raft Commit (Contracts + Metadata) 🗄️"]
      RS["Runi Scheduler (Raft-backed) 🧠"]
      LW["Launch Slices (Isolated Workers) 🚀"]
      ROD["Output Data 📦"]
      RIN --> RC --> RS --> LW --> ROD
    end
  end

  subgraph Failure_Recovery["⚡ Failure Recovery Flow (Crash Handling)"]
    subgraph MapReduce_Failure
      MFID["Input Data 📂"]
      MFR["Map Phase Running 🛠️"]
      MC["Map Node Crash 🛑"]
      MF["Job Fails Entirely ❌"]
      MR["Manual Restart Needed 🔄"]
      MFID --> MFR --> MC --> MF --> MR
    end

    subgraph RDD_Failure
      RFID["Input Data 📂"]
      RER["RDD Execution Running 🔄"]
      RCN["RDD Node Crash 🛑"]
      DR["Driver Attempts Lineage Recompute 🔁"]
      PR["Partial or Full Job Restart 🔄"]
      RFID --> RER --> RCN --> DR --> PR
    end

    subgraph Raft_Failure
      RFD["Input Data 📂"]
      SR["Slice Running 🚀"]
      RC["Raft Node Crash 🛑"]
      EL["Raft Detects Loss + Elects New Leader 🧠"]
      RE["Reschedule Slice Elsewhere ♻️"]
      CE["Continue Execution Seamlessly ✅"]
      RFD --> SR --> RC --> EL --> RE --> CE
    end
  end
{{< /mermaid >}}


## 🚀 How This Model Beats the Status Quo

### ✅ Compared to Apache Spark
| Spark (JVM) | Runink (Go + Linux primitives) |
|-------------|-------------------------------|
| JVM-based, slow cold starts | Instantaneous slice spawn using `exec` |
| Containerized via YARN/Mesos/K8s | No container daemon needed |
| Fault tolerance via RDD lineage/logs | Strong consistency via Raft |
| Needs external tools for lineage | Built-in governance and metadata |

---

### ✅ Compared to Kubernetes + Airflow
| Kubernetes / Airflow | Runink |
|----------------------|--------|
| DAGs stored in SQL, not consistent across API servers | DAGs submitted via Raft log, replicated to all |
| Task scheduling needs K8s Scheduler or Celery | Runi agents coordinate locally via consensus |
| Containers = overhead | Direct `exec` in a namespaced PID space |
| Secrets are environment or K8s Secret dependent | Raft-backed, RBAC-scoped Secrets Manager |
| Governance/logging external | Observability and lineage native and real-time |

---

## 🧠 Conclusion: Go + Linux internals + Raft = Data-Native Compute

Runink leverages **Raft consensus not just for fault tolerance, but as a foundational architectural choice**. It eliminates whole categories of orchestration complexity, state drift, and configuration mismatches by building from first principles — while offering **a single runtime that natively understands pipelines, contracts, lineage, and compute**.

If you’re designing a modern data platform — especially one focused on governance, and efficient domain isolation — Runink is a **radically integrated alternative** to the Kubernetes-centric model.