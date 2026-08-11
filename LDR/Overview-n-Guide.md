### Logical Data Replication (LDR) — Overview & Guide

### What is LDR?

**Logical Data Replication (LDR)** continuously replicates tables between an active *source* CockroachDB cluster and an active *destination* CockroachDB cluster. Both clusters can receive application reads and writes simultaneously (active-active setup), providing eventual consistency in the replicating tables. [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]

---

### Your Setup: Mumbai (Cluster A) ↔ Singapore (Cluster B)

- **Cluster A** — 2 nodes in Mumbai
- **Cluster B** — 2 nodes in Singapore
- Both clusters are active and can serve local traffic with **low, single-region write latency**
- LDR ensures tables on both clusters reach **eventual consistency**
- In a regional outage (e.g., Mumbai goes down), you can redirect all traffic to Singapore with **low downtime** [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]

---

### Use Cases

| Setup | Description |
|---|---|
| **Unidirectional** | Replicates from Cluster A (Mumbai) → Cluster B (Singapore) only |
| **Bidirectional** | Replicates both ways: Mumbai ↔ Singapore (active-active) |

- **High Availability**: Protect against datacenter/region failure while keeping single-region write latency [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]
- **Workload Isolation**: Run heavy jobs (backups, changefeeds) on one cluster without impacting the primary application cluster [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]

---

### Prerequisites

Before setting up LDR between your Mumbai and Singapore clusters: [[LDR Setup](https://www.cockroachlabs.com/docs/v24.3/set-up-logical-data-replication#before-you-begin)]

1. **Two separate CockroachDB self-hosted clusters** — LDR is **not** supported on CockroachDB Cloud.
2. **Full node-to-node connectivity** — Every node in Mumbai must be able to reach every node in Singapore and vice versa.
3. **Similar hardware profiles** — Both clusters should have a comparable number of nodes and overall size. Significant discrepancies may degrade performance.
4. **Matching table schemas** — Tables to be replicated must have **identical schema definitions** (excluding indexes) on both clusters.
5. **Certificate Authority access** — All nodes in each cluster need access to the Certificate Authority of the other cluster.
6. **External connections** — Set up external connections on the destination cluster to hold the connection URI for the source cluster.

---

### How to Set Up LDR

High-level steps: [[LDR Setup](https://www.cockroachlabs.com/docs/stable/set-up-logical-data-replication)]

1. Prepare both clusters with required settings, users, and privileges.
2. Set up external connection(s) on the destination cluster for the source URI.
3. Start LDR from the destination cluster using one of these SQL statements:
   - **`CREATE LOGICALLY REPLICATED`** — Automatically creates the table on the destination and performs a fast initial scan. Supports `unidirectional` or `bidirectional on` options.
   - **`CREATE LOGICAL REPLICATION STREAM`** — Use this if your table contains **user-defined types**; requires you to manually create the matching table first.
4. Monitor the LDR job status in the **DB Console**.

---

### ✅ Do's

- Use **fully qualified table names** in your LDR SQL statements.
- Use **bidirectional LDR** for active-active high availability between Mumbai and Singapore.
- Monitor LDR streams via the DB Console, Prometheus, or other observability tools.
- Use **multiple tables in one LDR job** for easier operations (pause/resume affects all tables together).
- Use **one table per LDR job** if you need table-level observability.
- Ensure **similar cluster sizing** between Mumbai and Singapore clusters. [[LDR Setup](https://www.cockroachlabs.com/docs/v24.3/set-up-logical-data-replication#before-you-begin)]

### ❌ Don'ts

- **Don't use LDR on CockroachDB Cloud** — it is only supported on self-hosted clusters. [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]
- **Don't use `CREATE LOGICALLY REPLICATED`** if your table contains **user-defined types** — use `CREATE LOGICAL REPLICATION STREAM` instead. [[LDR Setup](https://www.cockroachlabs.com/docs/stable/set-up-logical-data-replication)]
- **Don't have mismatched table schemas** between clusters — this will cause replication issues.
- **Don't expect strong consistency** — LDR provides **eventual consistency**, not transactional consistency. (For transactional consistency, use [Physical Cluster Replication](https://www.cockroachlabs.com/docs/stable/physical-cluster-replication-overview) instead.) [[LDR Overview](https://www.cockroachlabs.com/docs/stable/logical-data-replication-overview)]
- **Don't have significant hardware discrepancies** between clusters — this may result in degraded performance. [[LDR Setup](https://www.cockroachlabs.com/docs/v24.3/set-up-logical-data-replication#before-you-begin)]

---

> **Note:** LDR replicates at the **table level**, giving you flexibility to choose which specific tables to replicate — unlike PCR which replicates the entire cluster. [[LDR Blog](https://www.cockroachlabs.com/blog/logical-data-replication/#How-does-LDR-help-users-accomplish-these-goals?)]

<img width="1920" height="991" alt="image" src="https://github.com/user-attachments/assets/af250d93-683e-4afb-a7c9-a39a91c07517" />

---
### LDR Flow Structure — 2 Regional Clusters (Mumbai ↔ Singapore)

### Bidirectional LDR 

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/d3b9beb9-5de3-4bdc-8ce4-ee3db4d8a4af" />

---


#### Key Points

| Component | Mumbai (Cluster A) | Singapore (Cluster B) |
|---|---|---|
| **Role** | Source & Destination | Source & Destination |
| **Nodes** | 2 Nodes | 2 Nodes |
| **Traffic** | Active ✅ | Active ✅ |
| **Rangefeed** | Enabled ✅ | Enabled ✅ |
| **Replication** | Sends & Receives | Sends & Receives |

[[LDR Setup_CockroachDB Documentation](https://www.cockroachlabs.com/docs/stable/set-up-logical-data-replication)] [[Before You Begin](https://www.cockroachlabs.com/docs/stable/set-up-logical-data-replication#before-you-begin)]
