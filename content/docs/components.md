---
title: "Components"
weight: 11
---

# Runink Platform Components

This page describes the core building blocks of Runink — from the API server to slices — that make up the distributed data environment. Each component serves a purpose in ensuring secure, auditable, and high-performance pipeline execution.

## Components Table

| Component        | Role                                      | Location        |
|------------------|-------------------------------------------|-----------------|
| API Server       | Entry point, AuthN/Z, coordination        | Control Plane   |
| Identity Manager | OIDC/JWT validation and RBAC enforcement  | Control Plane   |
| Barn             | Raft-backed KV store                      | Control Plane   |
| Scheduler        | DAG-aware placement engine                | Control Plane   |
| Secrets Manager  | Encrypted secret storage and delivery     | Control Plane   |
| Governance Svc   | Lineage, quality, LLM annotations         | Control Plane   |
| Runi Agent       | Worker orchestrator (cgroup+namespace)    | Worker Node     |
| Runi Slice       | Executed unit of pipeline logic           | Worker Node     |
| Herd             | Tenant boundary and resource isolation    | System-wide     |
| Contracts        | Data validation and schema enforcement    | Contracts repo  |
| DSL Parser       | Converts `.dsl` to Go DAGs                | Build pipeline  |

---

## Runink Services

### 📘 Contract Engine

All data contracts (schemas) are defined in Go structs, with support for:

- Required/optional fields
- Type validation and coercion
- Golden testing and schema diffing
- Metadata annotations (e.g., PII, lineage tags)

💼 *Used in:* DSL `@contract`, golden tests, slice validation stages.


### ✍️ Feature DSL Parser

Parser and compiler for `.dsl` files.

- Converts scenario definitions into Go-based DAGs
- Enforces step ordering and contract compliance
- Attaches metadata for scheduling, RBAC, lineage

🔤 *Keywords:* `@step`, `@contract`, `@source`, `@sink`, `@affinity`, `@requires`.

---

## 🏃 Runi Agent

Daemon running on each **worker node**, responsible for execution.

- Registers with the control plane.
- Launches slices as **Go processes** within **cgroups** and **namespaces**.
- Sets up stdio pipes, config injection, and secrets access.
- Collects logs and exposes Prometheus metrics.

💡 *Design:* PID 1 in isolated namespace, manages ephemeral slices securely.

---

## ⚙️ Runi Slice

A **single unit of work** — a pipeline step — running in an isolated environment.

- Executed via `os.Exec` as a native Go binary.
- Enforces **herd-defined resource quotas** using **cgroups**.
- Receives config, secrets, and contracts.
- Reports lineage to Governance Service.

📦 *Properties:* Ephemeral, scoped, observable, auditable.

---

## 🧱 Barn (Cluster State Store)

A **Raft-backed KV store** providing durable, consistent cluster state.

- Stores pipeline definitions, slice metadata, herd configs, secrets, etc.
- Supports leader election and quorum for all control plane decisions.

🛡️ *Guarantees:* High availability, deterministic orchestration, and strong consistency.

---

## 🧰 Herd

Logical boundary for **multi-tenancy**, **quotas**, and **governance**.

- Maps to a namespace (network, user, mount, etc.).
- RBAC is scoped per herd.
- Resource quotas applied at the herd level.
- All metadata, secrets, and lineage are tagged with a herd context.

🔐 *Analogy:* Like Kubernetes namespaces but tighter and more secure.

---

## Herd Control Plane Services

### 📡 API Server 

The **entry point** for all client interactions (CLI, UI, and service integrations).

- Exposes REST/gRPC APIs secured via **OIDC/JWT**.
- Enforces **RBAC** and **herd scoping**.
- Forwards validated requests to:
  - State store (`Barn`)
  - Identity Manager
  - Scheduler
  - Secrets Manager
  - Governance Service

🔐 *Security:* Applies policies based on identity and herd-level permissions.

### 🧠 Identity & RBAC Manager

Responsible for identity resolution and access control.

- Validates JWT/OIDC tokens.
- Resolves user roles and herd membership.
- Provides per-herd scoped **RBAC policies**.

📘 *Location:* Can run co-located with the API server or standalone.

### 📅 Slice Scheduler

The component responsible for **task placement and orchestration**.

- Reads resource constraints from DSLs (`@requires`).
- Evaluates **herd quotas**, **affinities**, and **node health**.
- Determines **optimal slice placement**.
- Writes placements into `Barn`.

🧮 *Logic:* Constraint-solving over stateful inputs — affinity, quotas, node availability.

### 🔐 Secrets Manager

Handles secure secrets storage and delivery.

- Stores secrets in encrypted form (AES/GCM).
- Enforces access via **RBAC**.
- Slices receive secrets via `Runi Agent` during launch.

🗝️ *Design:* Secrets access scoped by herd and role, logged via Raft.

### 📊 Data Governance Service

Tracks **lineage**, **metadata**, and **annotations** for all slices.

- Stores rich metadata per run, stage, and contract hash.
- Supports querying for audit, compliance, and debugging.
- Can receive **LLM-based annotations**.

🔎 *Outputs:* Lineage graphs, quality reports, field-level annotations.

### 🔍 Observability Stack

Built-in support for:

- **Prometheus**: Metrics exposure via `/metrics`
- **Fluentd or stdout logs**: Structured JSON logs captured per slice
- **gRPC metadata reporting**: Trace context, tags, and result metadata

🧭 *Goal:* Enable deep pipeline inspection without needing external agents.

---

## 🔄 Pipes and Channels

Slices and agents use **pipes** (via `io.Pipe`, `os.Pipe`, `net.Pipe`) to transmit data and logs.

- Steps within a slice communicate via in-memory streams.
- No intermediate buffering — zero-copy, backpressure-safe.
- Logs are captured via stdout/stderr and piped to the agent.

🚰 *Benefits:* Stream processing, constant memory, no containers.

---
