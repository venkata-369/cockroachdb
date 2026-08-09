## PCR Tree Structure for Your Environment

Based on what we set up, here is your exact tree structure: [[PCR Technical Overview](https://www.cockroachlabs.com/docs/v25.2/physical-cluster-replication-technical-overview)]

---

### Singapore Cluster (Primary — ap-southeast-1)

```
CockroachDB Singapore Cluster (Primary)
├── system virtual cluster
│   └── service_mode = shared
│       (manages control plane + runs PCR producer job)
│
└── main virtual cluster
    └── service_mode = shared
        (manages data plane + runs application workloads)
```

---

### Mumbai Cluster (Standby — ap-south-1)

```
CockroachDB Mumbai Cluster (Standby)
├── system virtual cluster
│   └── service_mode = shared
│       (manages control plane + runs PCR consumer job)
│
└── main virtual cluster
    └── service_mode = none → shared (after failover)
        (receives replicated data from Singapore)
```

---

## Visual Diagram

![](https://www.cockroachlabs.com/docs/images/v24.2/physical-rep-to.png)

---

## What Each Component Does

| Component | Cluster | Role |
|---|---|---|
| `system` (Singapore) | Primary | Runs **producer job**, protects data with protected timestamps |
| `main` (Singapore) | Primary | Serves **application traffic**, source of replication |
| `system` (Mumbai) | Standby | Runs **consumer job**, initiates replication stream |
| `main` (Mumbai) | Standby | Receives replicated data from Singapore |

---

## Key Points

- The **system virtual cluster** manages the cluster's control plane and the replication of data [[PCR Technical Overview](https://www.cockroachlabs.com/docs/v25.2/physical-cluster-replication-technical-overview)]
- The **main virtual cluster** manages the data plane — user data and application workloads live here
- During replication, `main` on Mumbai has `service_mode = none` meaning it is **not serving traffic**
- After failover, `main` on Mumbai becomes `service_mode = shared` meaning it is **ready to serve traffic** ✅

---
## Virtual Cluster for Application Connectivity

The tree structure you saw represents a **named virtual cluster** (like `djs-ams-appconnet`) instead of the default `main`. This is how applications connect to a specific virtual cluster. [[CREATE VIRTUAL CLUSTER](https://www.cockroachlabs.com/docs/stable/create-virtual-cluster)]

---

## How to Create a Custom Named Virtual Cluster

On **Singapore (Primary)** system virtual cluster:

```sql
CREATE VIRTUAL CLUSTER djs-ams-appconnet;
```

Or during PCR replication setup:

```sql
CREATE VIRTUAL CLUSTER djs-ams-appconnet
  FROM REPLICATION OF system
  ON 'postgresql://replicator:repl123@10.30.2.151:26257/defaultdb?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=...';
```

---

## Start Service on the Virtual Cluster

```sql
ALTER VIRTUAL CLUSTER djs-ams-appconnet START SERVICE SHARED;
```

Verify:

```sql
SHOW VIRTUAL CLUSTERS;
```

Expected:
```
  id |  name               | data_state | service_mode
-----+---------------------+------------+---------------
   1 | system              | ready      | shared
   2 | djs-ams-appconnet   | ready      | shared
```

---

## Set as Default Cluster for Applications

```sql
SET CLUSTER SETTING server.controller.default_target_cluster = 'djs-ams-appconnet';
```

[[CREATE VIRTUAL CLUSTER](https://www.cockroachlabs.com/docs/stable/create-virtual-cluster)]

---

### How Applications Connect

#### Option 1: Default Connection (after setting default_target_cluster)

```
postgresql://user:password@10.30.2.151:26257/defaultdb?sslmode=verify-full&sslrootcert=certs/ca.crt
```

#### Option 2: Explicit Virtual Cluster Connection

```
postgresql://user:password@10.30.2.151:26257/defaultdb?options=-ccluster=djs-ams-appconnet&sslmode=verify-full&sslrootcert=certs/ca.crt
```

---

## How to Test Connection

```bash
cockroach sql \
  --url "postgresql://root@10.30.2.151:26257/defaultdb?options=-ccluster=djs-ams-appconnet&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

Then verify you are connected to the correct virtual cluster:

```sql
SHOW VIRTUAL CLUSTERS;
SELECT current_user();
```

---

### service_mode Explained

| service_mode | Meaning |
|---|---|
| `none` | Virtual cluster exists but **not serving traffic** (e.g., during replication) |
| `shared` | Virtual cluster is **active and serving application traffic** ✅ |

---

#### ⚠️ Important Note

The knowledge sources do not contain specific details about the exact `djs-ams-appconnet` configuration you saw in those documents. The name is just a **custom virtual cluster name** — the concept is the same as `main` but with a custom name chosen by the user. For full details refer to the [CockroachDB CREATE VIRTUAL CLUSTER docs](https://www.cockroachlabs.com/docs/stable/create-virtual-cluster).
