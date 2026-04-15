# Apache Spark — Software Architecture Recovery

> **Course Project | Software Architecture | Wuhan University**
> Prof. Peng Liang · liangp@whu.edu.cn

---

##  Team Members

| Name | Contribution So Far |
|------|---------------------|
| **Oussama** | Project setup, GitHub repository, architectural patterns analysis (Layered, Master-Worker), technical context documentation |
| **Lie** | Business context analysis, stakeholder identification for the Spark ecosystem |
| **Ming** | Component-and-connector structure recovery (Driver / Executor / Cluster Manager) |
| **Obada** | Module structure recovery, decomposition view of Spark Core subsystems |

>  **Project status: Midterm (Week 5)** — architecture recovery in progress. Final report due Week 11.

---

##  Project Overview

**Selected OSS Project:** [Apache Spark](https://github.com/apache/spark)

Apache Spark is a unified, open-source distributed computing engine for large-scale data processing. It is one of the most widely adopted big data frameworks in industry, making it an excellent subject for architecture recovery and analysis.

### Why We Chose Apache Spark

Apache Spark presents a rich and well-documented architecture spanning multiple structural dimensions — module decomposition, runtime component interaction, and deployment allocation. Its design decisions directly reflect the quality attributes discussed in the course (scalability, performance, fault tolerance, modifiability), and its real-world adoption across companies like Netflix, Alibaba, and Tencent gives us a concrete business context to analyze.

---

##  Architecture Recovery — Midterm Progress

### 1. What is Apache Spark's Software Architecture?

Following the definition from Bass et al. (*Software Architecture in Practice, 3rd Ed.*):

> *"The software architecture of a system is the set of structures needed to reason about the system, which comprise software elements, relations among them, and properties of both."*

We identified **three categories of architectural structures** in Spark:

---

### 2. Module Structures

**Decomposition View — Spark Core Subsystems**

```
Apache Spark
├── Spark Core
│   ├── SparkContext          (entry point, coordinates all operations)
│   ├── RDD                   (Resilient Distributed Dataset abstraction)
│   ├── DAGScheduler          (logical execution plan / stage graph)
│   ├── TaskScheduler         (physical task dispatch to executors)
│   └── BlockManager          (distributed in-memory/disk storage)
├── Spark SQL / DataFrame API
│   ├── Catalyst Optimizer    (logical → physical query planning)
│   └── Tungsten Engine       (code generation + memory management)
├── Structured Streaming      (real-time incremental processing)
├── MLlib                     (machine learning algorithms + pipelines)
└── GraphX                    (graph-parallel computation)
```

**Key module relationships:**
- Spark SQL, MLlib, Streaming, and GraphX all **depend on** Spark Core
- Spark Core is isolated from higher-level libraries → supports **modifiability**
- The Catalyst Optimizer and Tungsten Engine are encapsulated within Spark SQL, hiding optimization details from callers (information hiding principle)

---

### 3. Component-and-Connector Structures

**Runtime Architecture: Master-Worker Pattern**

```
┌──────────────────────────────────────────────────┐
│                  Driver Program                  │
│   SparkContext ──► DAGScheduler                  │
│                ──► TaskScheduler                 │
└────────────────────────┬─────────────────────────┘
                         │  submit tasks via RPC
                         ▼
┌──────────────────────────────────────────────────┐
│              Cluster Manager                     │
│       (YARN / Kubernetes / Standalone)           │
└──────────┬────────────────────────┬──────────────┘
           │  allocate resources    │  allocate resources
           ▼                        ▼
┌──────────────────┐     ┌──────────────────┐
│   Executor 1     │     │   Executor 2     │  ...N Executors
│  ┌────┐ ┌────┐   │     │  ┌────┐ ┌────┐  │
│  │Task│ │Task│   │     │  │Task│ │Task│  │
│  └────┘ └────┘   │     │  └────┘ └────┘  │
│  BlockManager    │     │  BlockManager   │
└──────────────────┘     └─────────────────┘
```

| Component | Role |
|-----------|------|
| **Driver** | Hosts the user application; builds the DAG; coordinates execution |
| **Cluster Manager** | Allocates CPU/memory resources across worker nodes |
| **Executor** | Runs tasks and caches data; reports results back to Driver |
| **BlockManager** | Manages distributed in-memory and disk storage per node |

**Connectors:** RPC calls (via Netty), shuffle data transfer, heartbeat messages

---

### 4. Architecture in Context (Chapter 3)

#### 🔧 Technical Context

Spark's architecture is driven by core quality attribute requirements:

| Quality Attribute | Architectural Decision |
|-------------------|----------------------|
| **Performance** | In-memory RDD caching vs. Hadoop's repeated disk I/O |
| **Scalability** | Linear horizontal scaling — add Executor nodes without changing architecture |
| **Fault Tolerance** | RDD lineage graph → automatic recomputation of lost partitions |
| **Modifiability** | Pluggable Cluster Manager interface (YARN, Kubernetes, Standalone, Mesos) |

####  Business Context

Spark was created at UC Berkeley's AMPLab in 2009 and open-sourced in 2010 to address real industry pain points: batch jobs that took hours in Hadoop finishing in **minutes**. Key business drivers that shaped the architecture:

- Need for iterative ML algorithms (Hadoop MapReduce was unsuitable for multi-pass algorithms)
- Support for interactive SQL queries alongside batch pipelines
- Multi-tenant cluster sharing across engineering teams

####  Stakeholders

| Stakeholder | Primary Concern |
|-------------|----------------|
| Data Engineers | Performance, API usability (Python/Scala/SQL) |
| Platform / Infra Teams | Scalability, resource management, Kubernetes integration |
| ML Engineers | MLlib correctness, GPU support, deep learning integration |
| Database Administrators | Spark SQL correctness, ACID transactions (Delta Lake) |
| Apache PMC / Contributors | Modifiability, test coverage, backward compatibility |

---

### 5. Why is Spark's Architecture Important? (Chapter 2)

| Reason | Spark Example |
|--------|--------------|
| **Inhibits or enables quality attributes** | Immutable RDDs + lineage tracking enable fault tolerance without data replication |
| **Earliest design decisions have lasting impact** | The choice of in-memory computing (2009) still defines Spark's performance identity today |
| **Reasoning about change** | Netflix added Structured Streaming *without* modifying Spark Core — evidence of good module isolation |
| **Conway's Law** | Spark's module structure (SQL, MLlib, Streaming, GraphX) mirrors the separate working groups within the Apache Spark PMC |
| **Basis for training** | Spark's layered architecture and well-documented views serve as an onboarding reference for new contributors |

---

##  Architectural Patterns Identified (So Far)

| Pattern | Where in Spark | Purpose |
|---------|---------------|---------|
| **Layered Architecture** | Spark Core → SQL / MLlib / Streaming / GraphX | Separation of concerns, modifiability |
| **Master-Worker** | Driver + Executors | Distributed task execution at scale |
| **Shared-Data (Repository)** | BlockManager + shuffle service | Distributed data access across nodes |
| **Pipeline** | DAG of RDD transformations | Lazy evaluation + query optimization |

---

##  Repo Structure

```
Software-Architecture/
├── README.md                   ← This file (midterm progress)
├── ARCHITECTURE_PATTERNS.md    ← Detailed pattern analysis
└── /diagrams/                  ← Architecture diagrams (coming in final)
```

---

##  Project Timeline

| Milestone | Week | Status |
|-----------|------|--------|
| Team formation + project proposal | Week 1–2 | ✅ Done |
| Architecture recovery (module + C&C structures) | Week 3–4 | ✅ Done |
| **Midterm presentation** | **Week 5** | ✅ Done |
| Allocation / deployment view | Week 6–7 | 🔄 In progress |
| Quality attribute scenario analysis (ATAM-style) | Week 8–9 | ⏳ Upcoming |
| Earliest design decisions documentation | Week 9–10 | ⏳ Upcoming |
| **Final presentation + report submission** | **Week 11** | ⏳ Upcoming |

---

##  References

- Bass, L., Clements, P., & Kazman, R. (2012). *Software Architecture in Practice* (3rd ed.). Addison-Wesley.
- Zaharia, M. et al. (2010). *Spark: Cluster Computing with Working Sets*. HotCloud '10.
- Apache Spark official documentation: https://spark.apache.org/docs/latest/
- Apache Spark GitHub repository: https://github.com/apache/spark
