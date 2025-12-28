# 1. Databricks Platform & Core Architecture

## 1.1 Databricks Overview

### What is Databricks?

Databricks is a **cloud-native Data Intelligence Platform** that unifies:

- Data engineering
- Data warehousing (SQL analytics)
- Machine learning
- Governance

on top of **Apache Spark + Delta Lake**, delivered as a **managed service**.

Key characteristics:

- Built around **open data formats** (Parquet, Delta)
- Separates **storage and compute**
- Strong focus on **end-to-end lifecycle** (ingestion → analytics → ML → BI)

---

### Why Databricks vs Traditional Systems

#### vs Traditional Data Warehouses

| Traditional DW              | Databricks                     |
| --------------------------- | ------------------------------ |
| Proprietary storage formats | Open formats (Delta Lake)      |
| Schema-on-write only        | Schema enforcement + evolution |
| Expensive scaling           | Elastic, cloud-native          |
| SQL-only                    | SQL + Python + Scala + ML      |

Interview angle:

- Databricks eliminates **data duplication** between lake + warehouse.
- Same data serves **BI + ML**.

---

#### vs Hadoop

| Hadoop                 | Databricks        |
| ---------------------- | ----------------- |
| Heavy ops (YARN, HDFS) | Fully managed     |
| Batch-oriented         | Batch + Streaming |
| On-prem focused        | Cloud-native      |

Key interview phrase:

> Databricks removed Hadoop’s operational complexity while retaining Spark’s scale.

---

#### vs Spark Standalone

Spark standalone:

- You manage clusters, configs, upgrades
- No governance, no orchestration, no lineage

Databricks:

- Managed Spark runtimes
- Built-in jobs, monitoring, security, SQL, MLflow

---

### Databricks as a **Data Intelligence Platform**

Interviewers increasingly expect this framing (2024+).

“Data Intelligence” =
**Data + AI + Governance + Cost Control**

Key pillars:

- Lakehouse (Delta Lake)
- Photon (performance)
- Unity Catalog (governance – later topic)
- AI-assisted analytics (AI/BI, ML integration)

---

## 1.2 Lakehouse Architecture

### Definition

A **Lakehouse** is an architecture that:

- Uses **data lakes as the single source of truth**
- Adds **warehouse-like guarantees** directly on the lake

---

### Core Principles

- Open storage formats
- Decoupled compute
- Transactional guarantees
- Unified analytics + ML

---

### How Lakehouse Combines Both Worlds

#### Data Lake Traits

- Object storage (S3 / ADLS / GCS)
- Cheap, scalable
- Open formats (Parquet)

#### Data Warehouse Traits

- ACID transactions
- Indexing / metadata
- Performance optimization
- Governance

Databricks Lakehouse =
**Delta Lake + Optimized Spark execution**

---

### Role of Delta Lake

Delta Lake is **non-negotiable** in interviews.

Delta provides:

- **ACID transactions** on object storage
- Schema enforcement & evolution
- Time travel
- Efficient metadata (transaction log)

Without Delta:

- Lakehouse does not exist
- Spark reads are eventually consistent
- No reliable incremental pipelines

Key interview soundbite:

> Delta Lake is the foundation that makes the Lakehouse reliable.

---

## 1.3 Control Plane vs Data Plane

### Control Plane

Managed by Databricks.

Includes:

- Workspace metadata
- Notebooks, repos
- Jobs orchestration
- Cluster definitions
- User & group management

**No customer data** stored here.

---

### Data Plane

Runs in **customer’s cloud account**.

Includes:

- Compute (VMs)
- Customer data
- Object storage
- Network traffic

Databricks deploys compute **inside your VPC/VNet**.

---

### Security Implications (Very High Yield)

- Databricks **cannot access your raw data**
- Data never leaves your cloud account
- Control plane compromise ≠ data access

Interview pitfall:

- Saying Databricks “hosts your data” → **incorrect**

---

## 1.4 Compute Architecture

### Databricks Compute Types

#### 1. All-Purpose Clusters

- Interactive
- Multi-user
- Used for:

  - Notebooks
  - Ad-hoc analysis
  - Development

Trade-off:

- More expensive
- Long-running

---

#### 2. Job Clusters

- Ephemeral
- One job run = one cluster lifecycle
- Used for:

  - Production pipelines
  - ETL jobs

Interview expectation:

> Prefer job clusters for production due to isolation + cost efficiency.

---

#### 3. SQL Warehouses

- Optimized for SQL
- Used by:

  - DBSQL
  - Dashboards
  - BI tools

Variants covered later (Serverless / Pro / Classic).

---

### Access Modes

#### Single User

- One user identity
- Required for:

  - Unity Catalog write access (non-shared)

- Best isolation

---

#### Shared

- Multiple users
- User isolation via Spark
- Some features restricted

---

#### No Isolation

- Legacy
- Least secure
- Generally discouraged

Interview note:

- Unity Catalog **restricts access modes intentionally**.

---

### Cluster Permissions

Controls:

- Who can attach
- Who can restart
- Who can terminate

Used to:

- Prevent accidental cluster misuse
- Enforce least privilege

---

### Cluster Policies

Declarative rules:

- Limit instance types
- Enforce auto-termination
- Control DBU cost

Common interview use case:

> Prevent developers from creating oversized clusters.

---

### Instance Pools

- Pre-warmed VMs
- Faster startup
- Lower cost for frequent jobs

Trade-off:

- Idle pool capacity costs money

---

## 1.5 Serverless Compute

### Serverless SQL Warehouses

- Fully managed
- Instant startup
- Auto-scaling
- No cluster visibility

Best for:

- BI
- Dashboards
- Ad-hoc SQL

---

### Serverless Notebooks

- No cluster management
- Elastic compute
- Ideal for exploration

Limitations:

- Less control
- Not all Spark configs exposed

---

### Interview Angle

Serverless =
**Operational simplicity over configurability**

---

## 1.6 Photon Engine

### What is Photon?

- Vectorized execution engine
- Written in **C++**
- Replaces parts of Spark execution engine

---

### When Photon Is Used

- SQL workloads
- DataFrame APIs
- Automatic (no code change)

Not used for:

- RDD-based logic
- Some UDF-heavy workloads

---

### Performance Implications

- 2–10x faster SQL
- Lower DBU cost for same workload
- Better CPU cache utilization

Interview trap:

- Photon is **not a separate cluster**
- Photon is **not optional at runtime** (auto-enabled when supported)

---

## 1.7 Databricks Roles & Administration

### Account Console

Account-level management:

- Users
- Workspaces
- Identity federation
- Unity Catalog metastore

---

### Workspace vs Account Level

| Workspace | Account           |
| --------- | ----------------- |
| Notebooks | Users             |
| Jobs      | Metastores        |
| Clusters  | Cloud credentials |

Interview emphasis:

> Governance lives at account level, not workspace level.

---

### Databricks Roles & Personas

- Data Engineers
- Analytics Engineers
- Data Scientists
- Platform Admins

Each persona maps to:

- Different compute
- Different access modes
- Different tools

---

### Identity Federation Basics

- SSO via:

  - Azure AD
  - Okta
  - IAM federation

- Centralized identity
- Enables auditability

---

### Interview Red Flags to Avoid

- Confusing control plane with data plane
- Treating Databricks as a proprietary warehouse
- Ignoring Delta Lake’s role
- Overusing all-purpose clusters in production
