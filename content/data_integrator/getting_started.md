---
title: "Getting Started"
weight: 90
---

# Getting Started with Runink

Welcome to **Runink**! This quick-start guide will help you get up and running with Runink to effortlessly build, test, and run data pipelines.

---

## 🚀 **1. Installation**

Make sure you have Go installed (v1.20 or later). Then install Runink:

```bash
go install github.com/runink/runink@latest
````

Ensure `$GOPATH/bin` is in your `$PATH`.

---

## 🛠 **2. Initialize Your Project**

Create a new Runink project in seconds:

```bash
runi init myproject
cd myproject
```

This command generates:

* Initial Go module
* Sample contracts
* Example `.dsl` files
* Golden file tests
* Dockerfile and CI/CD configs

---

## 📋 **3. Explore the Project Structure**

Your project includes:

```
myproject/
├── bin/                  -> CLI
├── contracts/            -> Schema contracts and transformation logic on Go structs
├── features/             -> Scenarios definitions for each feature from the `.dsl` files
├── golden/               -> Golden files used in regression testing with examples and synthetic data
├── dags/                 -> Generated DAG code from the contracts and features to be executed by Runi
├── herd/                 -> Domain Service Control Policies (Herd context isolation)
├── barn/                 -> Runi Agent manager: cgroups, metrics, logs, gRPC control plane
├── docs/                 -> Markdown docs, examples, use cases, and playbooks
└── .github/workflows/    -> CI/CD and test pipelines
```

---

## ⚙️ **4. Compile and Run Pipelines**

Compile your first pipeline:

```bash
runi compile --scenario features/example.dsl --out pipeline/example.go --herd my-data-herd
```

Execute a scenario:

```bash
runi run --scenario features/example.dsl --herd my-data-herd
```

---

## ✅ **5. Test Your Pipelines**

Use built-in golden testing to ensure correctness:

```bash
runi audit --scenario features/example.dsl --herd my-data-herd
runi synth --scenario features/example.dsl --herd my-data-herd
```

If logic changes are intentional, update golden files:

```bash
runi update --scenario features/example.dsl --update --herd my-data-herd
```

---

## 🔍 **6. Interactive REPL**

Explore and debug data interactively:

```bash
runi fetch --scenario features/example.dsl --herd my-data-herd
```

Example REPL commands:

```bash
load csv://data/input.csv
apply MyTransform
show
```

---

## 📚 **7. Next Steps**

* 📖 Learn the [Feature DSL Syntax](./feature-dsl.md) to write expressive data pipelines
* 🔎 Explore [Data Lineage & Metadata](./data-lineage.md) for auditability and reproducibility
* 📦 Understand [Schema & Contract Management](./schema-contracts.md) to ensure schema validation and drift detection

---

## 🚧 **Support & Community**

Need help or have suggestions?

* Open an [issue on GitHub](https://github.com/runink/runink/issues)
* Join our community discussions and get involved!

Let’s build secure, fast, and auditable pipelines — the Go-native way, with Runink.