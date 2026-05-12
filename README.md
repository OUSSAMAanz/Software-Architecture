# Apache Spark — Software Architecture Recovery

> **Course Project | Software Architecture | Wuhan University**
> Prof. Peng Liang · liangp@whu.edu.cn

---

## Team Members

| Name | Contribution So Far |
|------|---------------------|
| **Oussama** | Project setup, GitHub repository, architectural patterns analysis (Layered, Master-Worker), technical context documentation |
| **Lie** | Business context analysis, stakeholder identification for the Spark ecosystem |
| **Ming** | Component-and-connector structure recovery (Driver / Executor / Cluster Manager) |
| **Obada** | Module structure recovery, decomposition view of Spark Core subsystems |

> **Project status: Week 11** — architecture recovery complete. Final report and presentation ready for submission.

---

## 1. Business Context

Apache Spark was created at UC Berkeley's AMPLab in 2009 and open-sourced in 2010. It was designed to solve a critical industry pain point: large-scale data processing jobs that took **hours in Hadoop MapReduce** could be completed in **minutes** with in-memory computing.

### Why Spark Was Built

By 2009, Hadoop MapReduce dominated big data processing, but it had fundamental limitations:

- **Iterative algorithms** (machine learning, graph algorithms) required reading/writing disk on every pass — making them extremely slow
- **Interactive queries** were impractical because each SQL query triggered a new MapReduce job with full disk I/O
- **Multi-step pipelines** had to write intermediate results to HDFS between stages, adding hours of overhead

Spark's in-memory RDD abstraction solved all three problems with a single unified engine.

### Industry Adoption and Impact

Spark became one of the most widely adopted open-source projects in the big data ecosystem. Key adopters include:

| Company | Use Case |
|---------|----------|
| **Netflix** | Personalization pipelines, A/B test analysis |
| **Alibaba** | Large-scale e-commerce analytics, Taobao recommendation |
| **Tencent** | Social graph processing, WeChat data pipelines |
| **Uber** | Real-time trip pricing, fraud detection (Structured Streaming) |
| **NASA** | Scientific data processing at petabyte scale |

### Stakeholders

| Stakeholder | Primary Concern |
|-------------|----------------|
| Data Engineers | Performance, API usability (Python / Scala / SQL) |
| Platform / Infra Teams | Scalability, resource management, Kubernetes integration |
| ML Engineers | MLlib correctness, GPU support, deep learning integration |
| Database Administrators | Spark SQL correctness, ACID transactions (Delta Lake) |
| Apache PMC / Contributors | Modifiability, test coverage, backward compatibility |

---

## 2. Project Overview

**Selected OSS Project:** [Apache Spark](https://github.com/apache/spark)

Apache Spark is a unified, open-source distributed computing engine for large-scale data processing. It provides a consistent API across batch processing, SQL queries, streaming, machine learning, and graph computation — all on top of a single distributed execution engine.

### Why We Chose Apache Spark

Apache Spark presents a rich and well-documented architecture spanning multiple structural dimensions — module decomposition, runtime component interaction, and deployment allocation. Its design decisions directly reflect the quality attributes discussed in the course (scalability, performance, fault tolerance, modifiability), and its real-world adoption across companies like Netflix, Alibaba, and Tencent gives us a concrete business context to analyze.

---

## 3. Quality Attributes (Architectural Drivers)

The following quality attributes are the primary architectural drivers of Spark's design. Every major structural decision in Spark can be traced back to one or more of these.

| Quality Attribute | Scenario | Architectural Response |
|---|---|---|
| **Performance** | A 10-iteration ML training job must not re-read input data from disk on each iteration | RDD in-memory caching: after the first computation, partitions are stored in executor memory and reused |
| **Scalability** | A cluster must grow from 10 to 1000 nodes without changing the application code | Master-Worker architecture: Driver submits tasks to Cluster Manager, which allocates Executors dynamically; no code changes needed |
| **Fault Tolerance** | An Executor crashes mid-job; the pipeline must recover without restarting from scratch | RDD lineage graph: the DAGScheduler re-derives lost partitions by replaying only the failed partition's computation chain |
| **Modifiability** | Netflix wants to add Structured Streaming without breaking existing Spark SQL or MLlib code | Layered module structure: Streaming is a separate subsystem built on top of Spark Core; changes to Streaming do not propagate to Core |
| **Portability** | The same Spark job must run on YARN in production and on a laptop in Standalone mode | Pluggable Cluster Manager interface: YARN, Kubernetes, Mesos, and Standalone all implement the same `ClusterManager` abstraction |
| **Interoperability** | Data scientists want to write Spark jobs in Python, Scala, Java, or R | Language-agnostic API layer: PySpark, Scala API, SparkR all translate to the same internal RDD/DataFrame operations |
| **Resource Efficiency** | Multiple teams share the same cluster; resources must be isolated between jobs | Dynamic resource allocation: Executors are requested and released per-job by the Cluster Manager |

---

## 4. Architecture Recovery

### 4.1 Module Structure (Decomposition View)

The following diagram shows the **static decomposition** of Spark's subsystems and their dependencies.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Apache Spark                                     │
│                                                                          │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │   Spark SQL   │  │   Structured  │  │    MLlib    │  │  GraphX   │  │
│  │  / DataFrame  │  │   Streaming   │  │             │  │           │  │
│  │               │  │               │  │             │  │           │  │
│  │ ┌───────────┐ │  │ ┌───────────┐ │  │ Algorithms  │  │  Pregel   │  │
│  │ │ Catalyst  │ │  │ │ Micro-    │ │  │ Pipelines   │  │  API      │  │
│  │ │ Optimizer │ │  │ │ Batch Eng.│ │  │ Evaluators  │  │           │  │
│  │ └───────────┘ │  │ └───────────┘ │  │             │  │           │  │
│  │ ┌───────────┐ │  │               │  │             │  │           │  │
│  │ │ Tungsten  │ │  │               │  │             │  │           │  │
│  │ │ Engine    │ │  │               │  │             │  │           │  │
│  │ └───────────┘ │  │               │  │             │  │           │  │
│  └───────┬───────┘  └───────┬───────┘  └──────┬──────┘  └─────┬─────┘  │
│          │                  │                  │               │        │
│          └──────────────────┴──────────────────┴───────────────┘        │
│                                      │ all depend on                    │
│                       ┌──────────────▼──────────────┐                   │
│                       │         Spark Core           │                   │
│                       │                              │                   │
│                       │  SparkContext                │                   │
│                       │  RDD (abstraction)           │                   │
│                       │  DAGScheduler                │                   │
│                       │  TaskScheduler               │                   │
│                       │  BlockManager                │                   │
│                       │  MemoryManager               │                   │
│                       └──────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────────┘

KEY DEPENDENCY RULES:
  Spark Core      ──► no dependency on higher-level libraries
  Spark SQL       ──► depends on Spark Core only
  Streaming       ──► depends on Spark Core only
  MLlib           ──► depends on Spark Core; optionally uses Spark SQL
  GraphX          ──► depends on Spark Core only
  Catalyst/Tungsten ► internal to Spark SQL (information hiding)
```

**Key module relationships:**
- Spark SQL, MLlib, Streaming, and GraphX all **depend on** Spark Core — never on each other
- Spark Core is fully isolated from higher-level libraries → enables independent evolution of each subsystem
- The Catalyst Optimizer and Tungsten Engine are encapsulated within Spark SQL, hiding query optimization complexity from callers
- This structure reflects Conway's Law: each subsystem corresponds to a separate Apache Spark working group / PMC subteam

---

### 4.2 Component-and-Connector Structure (Runtime View)

**Master-Worker Pattern — Driver / Cluster Manager / Executor**

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Program                           │
│                                                                 │
│  User App Code                                                  │
│       │                                                         │
│       ▼                                                         │
│  SparkContext ──► DAGScheduler ──► Stage 1 ──► Stage 2 ──► ... │
│                        │                                        │
│                        ▼                                        │
│                  TaskScheduler                                  │
│                        │                                        │
└────────────────────────┼────────────────────────────────────────┘
                         │  submit tasks via RPC (Netty)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Cluster Manager                             │
│              (YARN / Kubernetes / Standalone)                   │
│                                                                 │
│  Resource allocation: CPU cores + memory per Executor           │
└──────────┬──────────────────────────────────┬───────────────────┘
           │ launch                           │ launch
           ▼                                  ▼
┌──────────────────────┐          ┌──────────────────────┐
│      Executor 1      │          │      Executor 2      │   ... N
│  ┌──────┐ ┌──────┐  │          │  ┌──────┐ ┌──────┐  │
│  │ Task │ │ Task │  │          │  │ Task │ │ Task │  │
│  └──────┘ └──────┘  │          │  └──────┘ └──────┘  │
│  BlockManager (cache)│◄────────►│  BlockManager (cache)│
└──────────────────────┘  shuffle └──────────────────────┘
         │  results via RPC                │
         └──────────────┬─────────────────┘
                        ▼
                   Driver collects
                   final output
```

| Component | Role |
|-----------|------|
| **Driver** | Hosts the user application; builds the DAG; coordinates execution |
| **SparkContext** | Entry point; connects to Cluster Manager; broadcasts config |
| **DAGScheduler** | Converts RDD lineage into stages; handles fault recovery |
| **TaskScheduler** | Maps stages to physical tasks; dispatches to Executors |
| **Cluster Manager** | Allocates CPU/memory resources across worker nodes |
| **Executor** | Runs tasks; caches partitions in memory; reports results to Driver |
| **BlockManager** | Per-node storage manager; serves cached partitions to other nodes |

**Connectors:** RPC calls via Netty (task dispatch, results), shuffle data transfer (between Executors), heartbeat messages (Executor → Driver every N seconds)

---

### 4.3 Architecture in Context

Following Bass et al. (*Software Architecture in Practice, 3rd Ed.*), a complete architecture recovery must address the system's context — the factors outside the system that shape its design.

**Technical Context:**

| External System | Interface | Purpose |
|---|---|---|
| HDFS / S3 / GCS | Hadoop FileSystem API | Input data source and output sink |
| YARN / Kubernetes | Cluster Manager API | Resource allocation and container lifecycle |
| Hive Metastore | JDBC / Thrift | Schema and table metadata for Spark SQL |
| Jupyter / Zeppelin | REST (Spark Connect) | Interactive notebook interfaces |
| Delta Lake / Iceberg | Data source API | ACID transaction support on top of Spark |

---

## 5. Key Architecture Design Decisions

### Decision 1 — In-Memory RDD Abstraction over Disk-Based MapReduce

**Context:** Iterative ML algorithms and interactive queries require reading the same data multiple times. Hadoop MapReduce writes intermediate results to HDFS between every stage.

**Decision:** Introduce the RDD (Resilient Distributed Dataset) as an immutable, fault-tolerant in-memory abstraction. Partitions are computed once and cached in executor memory for reuse.

**Rationale:** Eliminates repeated disk I/O for iterative workloads. Benchmarks showed 10–100× speedup over MapReduce for ML workloads.

**Trade-off:** Memory pressure on executors. If the dataset exceeds available RAM, Spark spills to disk (slower) or drops cached partitions and recomputes them.

---

### Decision 2 — Lineage-Based Fault Tolerance over Data Replication

**Context:** Distributed systems must tolerate node failures. The traditional approach (HDFS, databases) replicates data to N copies.

**Decision:** RDDs track their full computation lineage (the sequence of transformations that produced them). When a partition is lost, it is recomputed from its parent partitions using the stored lineage graph — no data is replicated.

**Rationale:** Replication is expensive in memory and network bandwidth. Recomputation is cheap for most transformations. Eliminates the need for a separate fault-tolerance layer.

**Trade-off:** If the original data source is unavailable (e.g., HDFS node down), recomputation fails. Long lineage chains can make recovery slow for deeply nested pipelines.

---

### Decision 3 — Pluggable Cluster Manager Interface

**Context:** Different organizations use different cluster resource managers (YARN in enterprise Hadoop shops, Kubernetes in cloud-native environments, Standalone for local development).

**Decision:** Spark defines a `ClusterManager` interface that abstracts resource acquisition. YARN, Kubernetes, Mesos, and Standalone are all pluggable implementations.

**Rationale:** Decouples Spark's execution engine from any specific infrastructure. The same Spark application runs unmodified across all deployment environments.

**Trade-off:** The abstraction layer limits Spark's ability to use advanced, cluster-manager-specific features (e.g., Kubernetes pod affinity rules require custom integration work).

---

### Decision 4 — Lazy Evaluation with DAG-Based Optimizer (Catalyst)

**Context:** Users chain many transformations together (filter → map → groupBy → join). Executing them eagerly (one at a time) would be extremely inefficient.

**Decision:** Transformations are lazy — they build a logical DAG but do not execute until an action (collect, save, count) is called. The Catalyst optimizer rewrites the DAG before execution (predicate pushdown, join reordering, etc.).

**Rationale:** Enables whole-query optimization that is impossible with eager execution. Catalyst can push filters early, eliminate unnecessary shuffles, and fuse adjacent map operations.

**Trade-off:** Debugging is harder because errors surface at action time, not at the transformation that caused them. Stack traces can be difficult to interpret.

---

### Decision 5 — Unified API for Batch, Streaming, SQL, and ML

**Context:** Data engineering teams typically need batch pipelines, real-time streaming, SQL analytics, and ML training — historically requiring four different tools (Hadoop, Storm, Hive, Mahout).

**Decision:** Build all four workloads on top of the same Spark Core engine with a unified DataFrame/Dataset API. Structured Streaming is implemented as a micro-batch engine on top of Spark SQL.

**Rationale:** Eliminates operational overhead of managing multiple systems. Code sharing between batch and streaming pipelines. Single cluster serves all workloads.

**Trade-off:** A unified API inevitably makes trade-offs. Spark Streaming's micro-batch model introduces latency (100ms–1s) that true streaming systems (Flink, Kafka Streams) avoid.

---

## 6. Architectural Patterns Identified

### Pattern 1 — Layered Architecture

**Where:** Overall module structure

```
┌──────────────────────────────────────────────────┐
│  Application Layer  (user PySpark / Scala code)  │
└──────────────────────┬───────────────────────────┘
                       │ uses
┌──────────────────────▼───────────────────────────┐
│  Library Layer  (SQL / MLlib / Streaming / GraphX)│
└──────────────────────┬───────────────────────────┘
                       │ all built on
┌──────────────────────▼───────────────────────────┐
│  Core Layer  (RDD / DAGScheduler / TaskScheduler) │
└──────────────────────┬───────────────────────────┘
                       │ executes via
┌──────────────────────▼───────────────────────────┐
│  Infrastructure Layer  (Cluster Manager + Nodes)  │
└──────────────────────────────────────────────────┘
```

**Purpose:** Separation of concerns and modifiability. Netflix added Structured Streaming without modifying Spark Core — direct evidence of effective layering.

---

### Pattern 2 — Master-Worker

**Where:** Driver ↔ Cluster Manager ↔ Executors (runtime)

```
         MASTER                        WORKERS
   ┌──────────────┐            ┌──────────────────┐
   │    Driver    │──tasks────►│   Executor 1     │
   │              │            │  [Task][Task]    │
   │ DAGScheduler │◄──results──│  BlockManager    │
   │ TaskScheduler│            └──────────────────┘
   └──────┬───────┘
          │ resource request   ┌──────────────────┐
          ▼                    │   Executor 2     │
   ┌──────────────┐──launch───►│  [Task][Task]    │
   │   Cluster    │            │  BlockManager    │
   │   Manager    │            └──────────────────┘
   └──────────────┘
                               ┌──────────────────┐
                               │   Executor N...  │
                               └──────────────────┘
```

**Purpose:** Enables horizontal scalability. Adding Executor nodes increases throughput linearly without architecture changes.

---

### Pattern 3 — Pipeline (DAG of Transformations)

**Where:** RDD/DataFrame transformation chains evaluated by DAGScheduler

```
Input Data (HDFS)
      │
      ▼
  [textFile]        ← Stage 0: read partitions
      │
      ▼
  [filter]          ← Stage 0: narrow transformation (no shuffle)
      │
      ▼
  [map]             ← Stage 0: narrow transformation
      │ shuffle boundary
      ▼
  [groupByKey]      ← Stage 1: wide transformation (triggers shuffle)
      │
      ▼
  [reduceByKey]     ← Stage 1: aggregation
      │
      ▼
  [collect / save]  ← Action: triggers execution of all stages
```

**Purpose:** Lazy evaluation allows the Catalyst optimizer to reorder, merge, and prune operations before any data is processed.

---

### Pattern 4 — Shared-Data (Repository)

**Where:** BlockManager + Shuffle Service

```
  Executor 1              Executor 2              Executor 3
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│  BlockManager│◄──────►│  BlockManager│◄──────►│  BlockManager│
│  (cache +    │        │  (cache +    │        │  (cache +    │
│   shuffle)   │        │   shuffle)   │        │   shuffle)   │
└──────┬───────┘        └──────┬───────┘        └──────┬───────┘
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               │
                    Distributed shared data
                    (RDD partitions, shuffle files)
                    accessible by any Executor
```

**Purpose:** Decouples data storage from computation. Executors can fetch cached partitions from any peer node, enabling efficient data reuse without centralizing storage.

---

## 7. Pattern Summary

| # | Pattern | Where in Spark | Purpose |
|---|---------|---------------|---------|
| 1 | Layered Architecture | Core → SQL / MLlib / Streaming / GraphX | Separation of concerns, modifiability |
| 2 | Master-Worker | Driver + Cluster Manager + Executors | Distributed task execution at scale |
| 3 | Pipeline (DAG) | RDD transformation chains → DAGScheduler | Lazy evaluation + query optimization |
| 4 | Shared-Data (Repository) | BlockManager + shuffle service | Distributed data access across nodes |

---

## 8. Repo Structure

```
Software-Architecture/
├── README.md                   ← This file (full architecture recovery)
├── ARCHITECTURE_PATTERNS.md    ← Detailed pattern analysis
└── /diagrams/                  ← Architecture diagrams (coming in final)
```

---

## 9. Project Timeline

| Milestone | Week | Status |
|-----------|------|--------|
| Team formation + project proposal | Week 1–2 | ✅ Done |
| Architecture recovery (module + C&C structures) | Week 3–4 | ✅ Done |
| **Midterm presentation** | **Week 5** | ✅ Done |
| Allocation / deployment view | Week 6–7 | ✅ Done |
| Quality attribute scenario analysis (ATAM-style) | Week 8–9 | ✅ Done |
| Earliest design decisions documentation | Week 9–10 | ✅ Done |
| **Final presentation + report submission** | **Week 11** | ✅ Done |

---

## 10. References

- Bass, L., Clements, P., & Kazman, R. (2012). *Software Architecture in Practice* (3rd ed.). Addison-Wesley.
- Zaharia, M. et al. (2010). *Spark: Cluster Computing with Working Sets*. HotCloud '10.
- Apache Spark official documentation: https://spark.apache.org/docs/latest/
- Apache Spark GitHub repository: https://github.com/apache/spark
