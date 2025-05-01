# Runink Architecture: Go/Linux Native Distributed Data Environment

Self-sufficient, distributed environment for orchestrating and executing data pipelines using Go and Linux primitives. This system acts as the cluster resource manager and scheduler (replacing Slurm), provides Kubernetes-like logical isolation and RBAC, integrates data governance features, and ensures robust security and observability. It aims for high efficiency by avoiding traditional virtualization or container runtimes like Docker. Define a self-sufficient, distributed environment for orchestrating and executing data pipelines using Go and Linux primitives, with enhanced metadata capabilities designed to support standard data governance (lineage, catalog) AND future integration of LLM-generated annotations.

Below one can find how each components coexist within the same cluster:

![](./user_flow.png)


### Runink Components

* **API Server / RBAC:** Enforces RBAC policies based on user/service account permissions within a target Herd.
* **Cluster State Store:** Explicitly stores Herd definitions and their associated resource quotas.
* **Scheduler:** Considers Herd-level quotas when making placement decisions.
* **Secrets Manager:** Access to secrets might be scoped by Herd.
* **Data Governance:** Metadata (lineage, annotations) can be tagged by or associated with the Herd it belongs to.
* **Runi Agent:** Receives the target Herd context when launching a slice and uses this information to potentially configure namespaces and apply appropriate cgroup limits based on Herd quotas.
* **Runi Slice:** A single instance of a pipeline step running as an isolated Worker Slice Process. Executes entirely within the logical boundary and resource constraints defined by its assigned Herd.
* **Runi Pipes:** Primarily used now for internal communication within the `Runi Agent` to capture logs/stdio from the `Runi Slice` it `exec`s, rather than for primary data transfer between steps.
* **Herd:** A logical grouping construct, similar to a Kubernetes Namespace, enforced via RBAC policies and potentially mapped to specific sets of Linux namespaces managed by Agents. Provides multi-tenancy and team isolation. Quotas can be applied per Herd.
* **Go Native & Linux Primitives:** Core components written in Go, directly leveraging cgroups, namespaces (user, pid, net, mount, uts, ipc), pipes, sockets, and `exec` for execution and isolation.
* **Self-Contained Cluster Management:** Manages a pool of physical or virtual machines, schedules workloads onto them, and handles node lifecycle.
* **Serverless Execution Model:** Users define pipelines and resource requests; Runink manages node allocation, scheduling, isolation, scaling (by launching more slices), and lifecycle. Users are subject to quotas managed via cgroups.
* **Security First:** Integrated identity (OIDC), RBAC, secrets management, network policies, encryption in transit/rest.
* **Data Governance Aware:** Built-in metadata tracking, lineage capture, and support for quality checks. With extension for storage/management of rich data annotations (e.g., from LLMs).
* **Rich Observability:** Native support for metrics (Prometheus) and logs (Fluentd).

---

# 🧠 Programming Approaches: Why They Power Runink

Runink’s architecture isn’t just Go-native — it’s intentionally designed around a few **low-level but high-impact programming paradigms**. These concepts are what let Runink outperform containerized stacks, enforce security without overhead, and keep pipelines testable, composable, and fast. Runink takes a radically different approach to pipeline execution than traditional data platforms — instead of running heavy containers, JVMs, or external orchestrators, Runink uses **Go-native workers**, Linux primitives like **cgroups and namespaces**, and concepts like **data-oriented design and zero-copy streaming** to deliver **blazing-fast, memory-stable, and secure slices**.

![](./concepts.png)

Below, we walk through the four core techniques and where they show up in Runink’s components.

---

## 🔄 1. **Functional Pipelines**
_"Like talking how your data flow over high-level functions."_

Runink's `.dsl` compiles to Go transforms that behave like **pure functions**: they take input (usually via `io.Reader`), apply a deterministic transformation, and emit output (via `io.Writer`). There's no shared mutable state, no side effects — just **clear dataflow**.

This makes pipelines:
- **Composable**: steps can be reused across domains
- **Testable**: golden tests assert input/output correctness
- **Deterministic**: behavior doesn't depend on cluster state

✅ **Why it matters:** It brings **unit testability** and **DAG clarity** to data pipelines — without needing a centralized scheduler or stateful orchestrator.

---
### 2. **Data-Oriented Design (DOD)**
_"Design for the CPU, not the developer."_

Instead of modeling data as deeply nested structs or objects, Runink favors **flat, contiguous Go structs**. This aligns memory layout with CPU cache lines and avoids heap thrashing. This is especially important for Runink’s slice execution and contract validation stages, where predictable access to batches of structs (records) matters. 

- Contracts are validated by scanning `[]struct` batches in tight loops.
- Pointers and indirection are minimized for GC performance.
- Contracts power both validation and golden test generation.
- Use slices of structs over slices of pointers to enable **CPU cache locality**.
- Align field access with columnar memory usage if streaming transforms run across many rows.
- Preallocate buffers in `Runi Agent`’s slice execution path to avoid GC churn.

---

### Core Idea:  
Layout memory for how it will be accessed, not how it's logically grouped. Runink’s slices often scan, validate, or enrich large batches of records — so struct layout, batching, and memory predictability **directly impact performance**.

### Apply DOD in Runink:
- Prefer **flat structs over nested ones** in contracts:
  ```go
  // Better
  type User struct {
    ID    string
    Name  string
    Email string
  }

  // Avoid: nested fields unless necessary
  type User struct {
    Meta struct {
      ID string
    }
    Profile struct {
      Name string
      Email string
    }
  }
  ```


### In `runi slice`:
- Use `sync.Pool` for reusable buffers (especially JSON decoding).
- Pre-size buffers based on contract hints (e.g., `maxRecords=10000`).
- Avoid interface{} — use generated structs via `go/types` or `go:generate`.

### Benefits: 
- Better **memory throughput**, fewer allocations, and **Go GC-friendliness** under load.
- Use **structs of arrays (SoA)** or `[]User` with preallocated slices in transformations.
- Minimize pointer indirection. Use value receivers and avoid `*string`, `*int` unless you need nil.
- Design transforms that operate in **tight loops**, e.g., `for _, rec := range batch`.
- Go structs are faster to iterate over than Python dictionaries or Java POJOs.
- Access patterns align with how CPUs fetch and cache data.
- Contract validation and transforms run over **preallocated `[]struct` batches**, not heap-bound objects.

💡 *For Python/Java devs:* Think of this like switching from `dicts of dicts` to **NumPy-like** flat arrays — but in Go, with static types and no GC spikes.

✅ **Why it matters:** You get **predictable memory use** and **cache-friendly validation** at slice scale — perfect for CPU-bound ETL or large-batch processing.

---

### 3. **Zero-Copy and Streaming Pipelines**
_"Avoid full in-memory materialization — process as the data flows."_

Instead of `[]record → transform → []record`, Runink pipelines follow **`stream → transform → stream`** — minimizing allocations and maximizing throughput. Avoid unnecessary data marshaling or full deserialization. Rely on:

- Transforms consume from `io.Reader` and emit to `io.Writer`.
- Stages communicate via `os.Pipe()`, `net.Pipe()`, or `chan Record` for intra-slice streaming.
- Only materialize records **when needed for validation or transformation**.
- Intermediate results never fully materialize in memory.

---

### Core Idea:
Instead of `[]Record -> Transform -> []Record`, operate on **streams of bytes or structs** using `io.Reader`, `chan Record`, or even UNIX pipes between stages.

### Runink Optimizations:
- Use **`io.Reader → Decoder → Transform → Encoder → io.Writer`** chain.
- Design step transforms like this:

  ```go
  func ValidateUser(r io.Reader, w io.Writer) error {
    decoder := json.NewDecoder(r)
    encoder := json.NewEncoder(w)

    for decoder.More() {
      var user contracts.User
      if err := decoder.Decode(&user); err != nil {
        return err
      }
      if isValid(user) {
        encoder.Encode(user)
      }
    }
    return nil
  }
  ```

- For multi-stage slices, use **`os.Pipe()`**:
  ```go
  r1, w1 := os.Pipe()
  r2, w2 := os.Pipe()

  go Normalize(r0, w1) // input -> step 1
  go Enrich(r1, w2)    // step 1 -> step 2
  go Sink(r2, out)     // step 2 -> sink
  ```

### Benefits:
- **Constant memory** even for massive datasets.
- **Backpressure**: If downstream slows down, upstream blocks — great for streaming (Kafka, etc.).
- Enables **DLQ teeing**: `tee := io.MultiWriter(validOut, dlqSink)`.
- Uses `io.Reader` / `io.Writer` rather than buffering everything in memory.
- Transforms run as **pipes between goroutines** — like UNIX but typed.
- Memory stays flat, predictable, and bounded — even for 10M+ record streams.

💡 *For pandas/Spark devs:* This is closer to **generator pipelines** or **structured stream micro-batches**, but with Go’s backpressure-aware channels and streaming codecs.


✅ **Why it matters:** You can process **unbounded streams or 100GB batch files** with a stable memory footprint — and gain built-in **backpressure and DLQ support**.

---

### 4. **Declarative Scheduling with Constraint Propagation**
_"Schedule via logic, not instructions."_
The `Herd` and `Runi Agent` coordination already benefits from Raft-backed state, but push it further with **affinity-aware, declarative scheduling**:

Runink doesn’t assign slices imperatively. It **solves** where to run things, based on:
- Isolation: `@herd("analytics")`
- Define **resource constraints** (e.g., `@requires(cpu=2, memory=512Mi, label=“gpu”)`) in `.dsl`.
- Placement: `@affinity(colocate_with="step:Join")`
- Propagate **slice placement decisions** through constraint-solving logic instead of imperative scheduling.
- Record constraints in the Raft-backed state store to enforce deterministic task placement.

You can build this as a small DSL-on-DSL layer (e.g. `@affinity(domain="finance", colocate_with="step:JoinUsers")`).

*Benefit:* Stronger **determinism**, **replayability**, and **multi-tenant safety**.

### Core Idea:
Model placement as a **set of constraints**: affinity, herd quota, GPU needs, tenant isolation, etc. Let the scheduler **solve** the placement rather than being told where to run.

### Runink DSL Extension:
In `.dsl`:

```gherkin
@step("RunLLMValidation")
@affinity(label="gpu", colocate_with="step:ParsePDFs")
@requires(cpu="4", memory="2Gi")
```

This can be compiled into metadata stored in the Raft-backed scheduler store.

### Scheduler Logic (Pseudo-Go):
```go
type Constraints struct {
  CPU       int
  Memory    int
  Affinity  string
  Colocate  string
  HerdScope string
}

func ScheduleStep(stepID string, constraints Constraints) (NodeID, error) {
  candidates := filterByHerd(constraints.HerdScope)
  candidates = filterByResources(candidates, constraints.CPU, constraints.Memory)
  if constraints.Colocate != "" {
    candidates = colocateWith(candidates, constraints.Colocate)
  }
  if constraints.Affinity != "" {
    candidates = matchLabel(candidates, constraints.Affinity)
  }
  return pickBest(candidates)
}
```

```plaintext 
[runi agent (PID 1 inside namespace)]
   |
   +--> Launch slice (Normalize step)
         |--> io.Pipe() Reader --> (Goroutine: Validate step)
         |--> io.Pipe() Reader --> (Goroutine: Enrich step)
         |--> Final Writer (sink to disk or message bus)
```

These constraints are stored in the Raft-backed `Barn` and evaluated by the scheduler. In this sense, all decisions are **Raft-logged**, making slice scheduling **auditable and replayable**.

💡 *If you're used to Kubernetes or Docker:* Think of slices as **ephemeral containers, but 10x faster** — no image pulls, no pod scheduling latency. No containers, no clusters — just data pipelines that behave like code.

✅ **Why it matters:** Runink achieves **multi-tenant safety, fault-tolerant execution, and reproducible placement** — without complex K8s YAML or retries.

---

### Summary Table

| **Approach**               | **Use In Runink**                                             | **Why It Powers Runink**                         |
|----------------------------|---------------------------------------------------------------|--------------------------------------------------|
| Functional Pipelines       | `.dsl` → Go transforms via `@step()`                          | Clear transforms, reusable logic, golden testing |
| Data-Oriented Design       | Contract enforcement, slice internals                         | Memory locality, low-GC, CPU-efficient pipelines |
| Zero-Copy Streaming        | Slice-to-slice transport, pipe-to-pipe steps                  | Constant memory, streaming support, low latency  |
| Declarative Scheduling     | Herd quotas + slice placement affinity in `.dsl` raft store   | Deterministic, fair, replayable orchestration    |

---

### 5. **Golden Testing and Contract-Aware Execution**

All pipelines are backed by **Go structs as schema contracts**:

- Versioned, diffable, validated at runtime
- Pipeline steps are tested using **golden output** and **synthetic data generators**
- Runink can **detect schema drift** and route invalid records to DLQ

💡 *This gives you the safety of DBT with the flexibility of code-first transforms.*


## 📈 Technology Benchmark: Go + Raft + Linux Primitives vs. JVM + Containerized Stacks

| **Dimension**                 | **Traditional (JVM / Container / DataFrame)**     | **Runink (Go / FP Pipelines / Raft)**                             | **Gains**                                                                                                                                                                         |
| ----------------------------------- | ------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Language Runtime**          | Java Virtual Machine                                    | Go runtime                                                              | •**Lower startup latency** (milliseconds vs. seconds) • **Smaller memory footprint** per process (tens of MBs vs. hundreds of MBs)                                        |
| **Programming Paradigm**      | OOP (classes, inheritance, heavy type hierarchies)      | Functional-ish pipelines in Go (pure functions, composable stages)      | •**Simpler abstractions** (no deep inheritance)  • **Easier reasoning** about data flow  • **Higher testability** via pure stage functions                         |
| **Data Processing Model**     | Row-based DataFrames                                    | Channel-based streams of structs (Go channels → functional transforms) | •**Constant memory** for streaming (no full in-memory tables)  • **Back-pressure** and windowing naturally via channels  • **Composable, step-by-step transforms** |
| **Isolation & Sandboxing**    | Containers + VM overhead                                | cgroups + namespaces + Go processes                                     | •**0.5–1 ms** slice startup vs. **100 ms+** container spin-up  • **Fine-grained resource limits** per slice  • **No dockerd overhead**                      |
| **Inter-Stage Communication** | Shuffle via network storage or distributed file systems | In-memory pipes, UNIX sockets, or gRPC streams within Go                | •**<1 ms** handoff latency  • **Zero-copy** possible with `io.Pipe`  • **Strong type safety** end-to-end                                                         |
| **Distributed Consensus**     | External etcd or no coordination                        | Built-in Raft                                                           | •**Linearizable consistency** for control plane  • **Leader election** and automatic recovery  • **Atomic updates** across scheduler, secrets, and metadata        |
| **Multi-Tenancy & Security**  | K8s namespaces + complex RBAC, network policies         | Herd namespaces + cgroup quotas + OIDC/JWT + mTLS                       | •**Single-layer isolation**  • **Per-slice ephemeral UIDs** for zero-trust  • **Simpler, declarative Herd-scoped policies**                                        |
| **Observability & Lineage**   | External stacks                                         | Native agents + Raft-consistent metadata store                          | •**Always-consistent** lineage  • **Unified telemetry** (metrics, logs, traces)  • **Real-time auditability** without manual integration                           |
| **Deployment & Ops**          | Helm charts, CRDs, multi-tool CI/CD                     | Single Go binary + Raft cluster bootstrap                               | •**One artifact** for all services  • **Simpler upgrades** via rolling Raft restarts  • **Built-in health & leader dashboards**                                    |
| **Functional Extensibility**  | Plugins in Java/Python with JNI or UDFs (heavy)         | Go plugins or simple function registration                              | •**No cross-language bridges**  • **First-class Go codegen** from DSL  • **Easier on-the-fly step injection**                                                      |

---

### Key Takeaways

1. **Speed & Efficiency**Go’s minimal runtime and direct use of Linux primitives deliver sub-millisecond task launches and minimal per-slice overhead—vs. multi-second container or JVM startups.
2. **Deterministic, Fault-Tolerant Control**Raft gives Runink a single, consistent source of truth across all core services—eliminating split-brain and eventual-consistency pitfalls that plague layered stacks.
3. **Composable, Functional Pipelines**Channel-based, stage-oriented transforms allow lean, testable, streaming-first ETL—rather than bulk-loading entire tables in memory.
4. **Unified, Secure Multi-Tenancy**Herds + namespaces + cgroups + OIDC/JWT yield strong isolation and a simpler security model than stitching together K8s, Vault, and side-cars.
5. **Built-In Governance & Observability**
   Raft-backed metadata ensures lineage and schema contracts are always in sync—no external governance platform required.

By grounding Runink’s design in **Go**, **functional pipelines**, **Linux primitives**, and **Raft consensus**, you get a single, vertically integrated platform that outperforms and out-operates the conventional “JVM + containers + orchestration + governance” stack—delivering **predictable**, **low-overhead**, and **secure** data pipelines at scale.

## 🧠 Raft-Backed Architecture

At the heart of Runink's distributed control plane is Raft consensus, which ensures strong consistency and availability across multiple replicas in a multi-tenant setup. Here’s how Raft plays a crucial role:

### Cluster State Store (Barn):

*Raft ensures* consistency between multiple replicas of the state store, preventing conflicts and enabling data to be synchronized across different services. With critical data, such as pipeline definitions, herd configurations, secrets, and RBAC policies, is replicated and stored in the Barn, which is under Raft consensus.

*Writes and Reads*: All writes (e.g., task assignments, secrets, RBAC policies) go through the Raft leader to ensure consistency. If the leader fails, Raft elects a new leader to ensure continuous operation without data loss.

### Scheduler:

Uses Raft's consensus for task scheduling to ensure that nodes in the cluster agree on which worker slices should be assigned the next task.
Task assignments are synchronized across the cluster to avoid conflicts and duplications.

### Secrets Manager:

Managed via Raft to ensure that secrets are consistently available across all nodes, with encrypted storage and strict RBAC.
Guarantees that secrets are securely injected into Runi agents and worker slices at runtime, with access scoped per Herd.

### Data Governance & Lineage:

Lineage and metadata are tagged and tracked consistently across the cluster, with Raft-backed guarantees to prevent discrepancies in historical records.

### Resilience & High Availability:

Raft consensus enables Runink to maintain availability and consistency in the face of node failures by synchronizing state across multiple replicas.
The API Server, Scheduler, Secrets Manager, and other components benefit from Raft to ensure high availability, disaster recovery, and distributed state consistency.

With Raft integrated into Runink, the system can operate fault-tolerantly, ensuring that data across the entire platform remains consistent, even when network partitions or node failures occur. This guarantees that your data pipelines, metadata, secrets, and workload scheduling are managed reliably in any cloud-native or distributed environment

---

## User interaction

Runink operates with a Control Plane managing multiple Worker Nodes, each running a Runi Agent.

![](./user_interaction.png)


## Future LLM Integration

1. **Pipeline Definition:** A user defines a pipeline step specifically for LLM annotation. This step specifies the input data source (e.g., path on shared FS/MinIO), the target LLM (e.g., OpenAI model name or internal service endpoint), the prompt, and potentially the output format.
2. **Scheduling:** The `Scheduler` assigns this step to a `Runi Agent`. If targeting an internal LLM requiring specific hardware (GPU), the scheduler uses node resource information (reported by Agents) for placement.
3. **Execution:** The `Runi Agent` launches a `Worker Slice Process` for this step.
4. **Credentials:** The Worker Slice receives necessary credentials (e.g., OpenAI API key, MinIO access key) securely via the `Secrets Manager`.
5. **LLM Call:** The worker reads input data, constructs the prompt, calls the relevant LLM API (external or internal), potentially handling batching or retries.
6. **Metadata Persistence:** Upon receiving results, the worker extracts the annotations, formats them according to the `Data Governance Service` schema, and sends them via gRPC call to the service, linking them to the input data reference. It also reports standard lineage (input data -> LLM step -> annotations).
7. **Usage:** Downstream pipeline steps or external users can then query the `Data Governance Service` (via `API Server`) to retrieve these annotations for further processing, reporting, or analysis.
