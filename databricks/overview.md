# 01 Overview of Databricks

## ✅ 1. **What is Databricks?**

**Databricks** is a **cloud-native data analytics platform** built on **Apache Spark** that enables organizations to unify **data engineering, data science, machine learning (ML), and business intelligence (BI)** workloads in one place. It provides a **lakehouse architecture** — combining the flexibility of data lakes with the performance and management features of data warehouses.

#### Key Points

- It was founded by the original creators of **Apache Spark**, and extends it for cloud-scale analytics.
- Databricks powers ETL, ad-hoc analytics, ML model building, and business dashboards on _the same data platform_.
- Includes components such as **Delta Lake (transactional storage layer)** and **Databricks SQL** for BI.

---

## ✅ 2. **Why do we need Databricks?**

The landscape of enterprise data has changed:

- Data is **huge** (petabytes).
- Data is **diverse** (structured, semi-structured, unstructured).
- Users range from **engineers and data scientists to analysts and product owners**.

Databricks is needed because:

#### 🌟 1) **Unified Platform**

Traditionally analytic systems were siloed (data warehouse for BI, data lake for ML). Databricks provides one system that supports:

- **ETL/ELT**
- **Streaming & batch processing**
- **Machine learning**
- **SQL analytics and dashboards**
  All on one shared metadata and governance layer.

#### 🌟 2) **Scalability & Cost Efficiency**

It decouples **storage and compute**, letting companies pay only for processing when needed and use cheap cloud object storage for data.

#### 🌟 3) **Support for Diverse Workloads**

From **SQL reporting** to **Python/ML notebooks**, to **streaming ingestion**, Databricks supports the full data lifecycle in one platform.

#### 🌟 4) **Open Standards & No Vendor Lock-In**

Uses open formats like **Parquet/Delta**, and open projects (Apache Spark, Delta Lake, MLflow), so your data isn’t locked into a proprietary format.

---

## ✅ 3. **What is a Lakehouse?**

A **lakehouse** is a modern data architecture that **unifies the best of data lakes and data warehouses**.

#### At a high level:

- Like a **data lake**, it can store _all types_ of data — structured, semi-structured, unstructured — in open formats.
- Like a **data warehouse**, it supports **ACID transactions, schema enforcement, performance optimizations, governance, and BI queries.**

#### What a Lakehouse enables

- One **single source of truth** across analytics and AI workloads.
- **Remove redundant copies** between systems (no separate data warehouse + data lake pipelines).

---

## ✅ 4. **Challenges Traditional Platforms Face (and how Databricks solves them)**

#### 🚫 **1) Data Silos**

Traditional architectures often had separate:

- Data lakes (cheap, flexible storage)
- Data warehouses (structured analytics)
- Specialized tools for ML

This leads to duplicate data and complex ETL between systems.

📌 **Databricks Solution:** A single lakehouse, eliminating the need to move data between systems.

---

#### 🚫 **2) Lack of Governance & Quality in Data Lakes**

Raw data lakes lack:

- ACID transactions
- Schema enforcement
- Data quality checks
- Unified security controls

without these, lakes easily become **data swamps**.

📌 **Databricks Solution:**

- **Delta Lake layer** brings ACID, schema enforcement, time travel.
- **Unity Catalog** provides centralized fine-grained governance, access controls, and lineage.

---

#### 🚫 **3) Traditional Warehouses Aren’t Flexible for AI/ML**

Data warehouses are optimized for structured SQL but struggle with:

- Semi-structured/unstructured data (images, text, logs)
- Spark/ML workloads
  This creates friction for data scientists.

📌 **Databricks Solution:** Supports rich data types and analytical engines in one platform on open storage.

---

#### 🚫 **4) Slow Data Freshness**

In traditional architectures, data lakes write raw data → then ETL to warehouse → analytics, which creates **data staleness**.

📌 **Databricks Solution:** Direct analytics on the same governed lakehouse data, reducing delay and making data ready for analytics and BI sooner.

---

#### 🚫 **5) Cost & Complexity of Multiple Tools**

Managing multiple systems increases:

- Operational overhead
- Cost of storage and compute
- Engineering overhead on keeping data in sync

📌 **Databricks Solution:** Reduces operational overhead with one integrated platform and managed services on cloud.

---

## ✅ 5. **Data Lake vs Data Warehouse vs Lakehouse**

Here’s a simple comparison:

| Feature               | **Data Lake** | **Data Warehouse** | **Data Lakehouse**     |
| --------------------- | ------------- | ------------------ | ---------------------- |
| Data Type             | All (raw)     | Structured only    | All (raw + structured) |
| Cost                  | Low           | Higher             | Medium / Efficient     |
| Governance & ACID     | ❌            | ✔️                 | ✔️                     |
| Analytics Performance | ❌            | ✔️                 | ✔️                     |
| ML Support            | ✔️            | ❌ / Limited       | ✔️                     |
| BI Dashboards         | ❌ (hard)     | ✔️                 | ✔️                     |

### Quick Definitions

**Data Lake**

- A repository for _all kinds_ of raw data.
- Cheap and scalable, but lacks transactional guarantees and governance.

**Data Warehouse**

- A structured system optimized for **BI and SQL analytics**.
- Strong governance and performance but limited in data variety.

**Data Lakehouse**

- Merges the _best of both_ — open storage, governance, performance, and analytics in one place.

---

## 🧠 Final Summary

#### **Why Databricks matters**

- It solves **data complexity, governance, and analytics fragmentation** by unifying storage and compute in a managed, scalable cloud platform.
- It enables **BI, ML, and advanced analytics on the _same data platform_** saving cost and reducing engineering overhead.

#### **Lakehouse Advantages**

- **Single source of truth**
- **Open formats & multi-engine access**
- **ACID transactions**
- **Governance & lineage**
- **Support for real-time streaming and dashboards**

These capabilities are essential for **modern AI and analytics workloads** that traditional systems struggle to handle efficiently.
