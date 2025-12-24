# 02 Databricks Architecture & Core Roles

_Control Plane vs Data Plane · Account vs Workspaces · Metastores_

---

## 1️⃣ Concept Overview

### What is Databricks Architecture?

Databricks architecture defines **how compute, storage, governance, and user access are organized and isolated** in the Databricks Lakehouse Platform.

At a high level, Databricks is built on:

- **Cloud-native separation of control and data**
- **Multi-workspace isolation**
- **Centralized governance via Unity Catalog**

### Why This Architecture Exists

Traditional big data platforms struggled with:

- Tight coupling of compute and storage
- Weak isolation between teams
- Fragmented governance
- Manual security controls
- Poor multi-tenant scalability

Databricks architecture addresses these via:

- Control Plane vs Data Plane separation
- Account-level governance
- Centralized metadata and access control
- Elastic compute per workload

### Problems It Solves

- Secure multi-tenant analytics
- Scalable platform governance
- Centralized metadata & permissions
- Cloud-native cost control
- Enterprise-grade isolation

---

## 2️⃣ Core Architecture / How It Works

### High-Level Architecture Diagram

![Databricks High-Level Architecture](https://docs.databricks.com/aws/en/assets/images/architecture-c2c83d23e2f7870f30137e97aaadea0b.png)

---

### 🧠 Control Plane vs 💾 Data Plane

#### 🔹 Control Plane

Managed entirely by **Databricks (SaaS layer)**.

Responsible for:

- Workspace metadata
- Job orchestration
- Cluster management
- Notebooks, repos
- Unity Catalog metadata
- Authentication & authorization
- REST APIs

**Key characteristics**

- No customer data processed
- Runs in Databricks-managed cloud
- Multi-tenant
- Highly secured and audited

> ⚠️ Interview Tip: **Customer data NEVER flows through the control plane**

---

#### 🔹 Data Plane

Runs **inside the customer’s cloud account (AWS/Azure/GCP)**.

Responsible for:

- Spark compute (clusters, SQL warehouses)
- Data processing
- Reading/writing cloud storage
- Photon execution
- Delta Lake operations

**Key characteristics**

- Fully isolated per customer
- Uses customer VPC/VNet
- Data never leaves customer account
- IAM-based access to storage

---

### Control Plane vs Data Plane Summary

| Aspect         | Control Plane   | Data Plane        |
| -------------- | --------------- | ----------------- |
| Ownership      | Databricks      | Customer          |
| Contains Data  | ❌ No           | ✅ Yes            |
| Compute        | ❌              | ✅                |
| Metadata       | ✅              | ❌                |
| Networking     | Databricks SaaS | Customer VPC/VNet |
| Security Model | SaaS IAM + APIs | Cloud IAM         |

---

## 3️⃣ Account vs Workspace Architecture

### 🏢 Databricks Account

The **Account** is the **top-level administrative boundary**.

Responsible for:

- Identity federation (SCIM, SSO)
- Workspace creation
- Unity Catalog metastores
- Account-level groups
- Cross-workspace governance

> Think of Account as the **organization-wide control layer**

---

### 🧪 Databricks Workspace

A **Workspace** is an isolated environment where users:

- Run notebooks
- Create jobs
- Use clusters and SQL warehouses
- Develop pipelines

Each workspace:

- Has its own notebooks, jobs, clusters
- Can attach to **one Unity Catalog metastore**
- Is isolated from other workspaces

---

### Account → Workspace Relationship Diagram

![Account vs Workspace](https://docs.databricks.com/aws/en/assets/images/object-hierarchy-966dee8fa239626134ed6b7278beab2c.png)

---

### Key Rules

- One **Account** → many **Workspaces**
- One **Workspace** → one **Metastore**
- One **Metastore** → many **Workspaces**

---

## 4️⃣ Metastores (Unity Catalog)

### What Is a Metastore?

A **Metastore** is a **centralized metadata and governance layer** in Unity Catalog.

It stores:

- Catalogs
- Schemas
- Tables
- Views
- Functions
- Permissions & ownership
- Lineage metadata

---

### Why Metastores Exist

Legacy Hive Metastore issues:

- Workspace-scoped
- Weak security
- No lineage
- Hard to govern across teams

Unity Catalog metastores solve:

- Central governance
- Cross-workspace access
- Fine-grained permissions
- Auditing and lineage

---

### Metastore Architecture Diagram

![Unity Catalog Metastore](https://docs.databricks.com/aws/en/assets/images/object-model-40d730065eefed283b936a8664f1b247.png)

---

### Metastore Scope & Binding

- Created **at Account level**
- Bound to a **cloud region**
- Attached to multiple workspaces
- Uses a **managed or external storage location**

---

### Example: Metastore Creation (Conceptual)

```bash
# Metastore is created via account console or API
# Then assigned to workspaces
```

---

## 5️⃣ Key Features & Capabilities

### Architecture Highlights Interviewers Care About

- Strict Control/Data plane separation
- Customer-owned data plane
- Centralized governance via Unity Catalog
- Multi-workspace isolation
- Cloud IAM integration
- Cross-workspace metadata sharing

---

## 6️⃣ Common Design Patterns

### ✅ When to Use Multiple Workspaces

- Environment isolation (dev / test / prod)
- Team-level isolation
- Compliance requirements
- Cost attribution

### ❌ When NOT to Create Too Many Workspaces

- If governance is weak
- If data duplication increases
- If operational overhead outweighs benefits

---

### Recommended Enterprise Pattern

```
Account
 ├── Workspace: dev
 ├── Workspace: test
 └── Workspace: prod
       └── Shared Unity Catalog Metastore
```

---

## 7️⃣ Performance, Security, and Cost Considerations

### 🔐 Security

- Data plane uses customer IAM roles
- Unity Catalog enforces row/column-level security
- Control plane never sees raw data
- Private Link / VPC peering supported

### ⚡ Performance

- Photon runs in data plane
- Control plane latency does not affect query execution
- Metadata caching improves query planning

### 💰 Cost

- Compute costs driven by data plane
- Control plane costs included in Databricks pricing
- Workspace sprawl increases operational cost

---

## 8️⃣ Common Pitfalls & Misconceptions

> ⚠️ **Misconception:** Control plane processes customer data
> ✅ Reality: Only metadata and orchestration

> ⚠️ **Mistake:** One workspace per team without shared metastore
> ✅ Leads to governance fragmentation

> ⚠️ **Mistake:** Assuming Hive Metastore = Unity Catalog
> ✅ Hive is deprecated for new deployments

---

## 9️⃣ Interview-Focused Q&A

**Q1. Why does Databricks separate control and data planes?**
To ensure security, scalability, and cloud-native isolation.

**Q2. Can Databricks access customer data?**
No. Data stays in the customer’s cloud account.

**Q3. What is the role of an Account vs Workspace?**
Account manages governance; workspace runs workloads.

**Q4. Can one workspace use multiple metastores?**
No. One workspace attaches to only one metastore.

**Q5. Why is Unity Catalog account-scoped?**
To enable centralized, cross-workspace governance.

---

## 🔟 Quick Revision Summary

- Control Plane = metadata & orchestration
- Data Plane = compute & data processing
- Account = org-level governance
- Workspace = execution boundary
- Metastore = centralized metadata & permissions
- Unity Catalog is the **default and recommended** approach
- Data never leaves customer cloud

---

> 📌 **Interview Gold Line:** > _“Databricks achieves enterprise-grade governance by separating control and data planes, and centralizing metadata using account-scoped Unity Catalog metastores.”_
