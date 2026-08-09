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
