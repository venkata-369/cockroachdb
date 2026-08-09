### ✅ PCR is Fully Operational!

Replication is in **steady state**! Here is what the output means: [[PCR Monitoring CockroachDB Documentation](https://www.cockroachlabs.com/docs/v25.2/physical-cluster-replication-monitoring#sql-shell)]

| Field | Value | Status |
|---|---|---|
| `status` | `replicating` | ✅ Fully replicating |
| `replication_lag` | `00:00:15` | ✅ Excellent — only 15 seconds behind |
| `replicated_time` | `2026-08-09 04:41:00` | ✅ Advancing |
| `retained_time` | `2026-08-09 04:38:41` | ✅ Failover window active |
| `failover_time` | `NULL` | ✅ Normal — no failover initiated |

---

### ✅ Complete PCR Setup Summary

| Cluster | Region | Role | Status |
|---|---|---|---|
| Singapore (crdb-node8, crdb-node9) | ap-southeast-1 | **Primary** | ✅ Running |
| Mumbai (crdb-node3, crdb-node4) | ap-south-1 | **Standby** | ✅ Replicating |

---

### ⏭️ What Next

1. **Test Failover** — promote Mumbai standby to primary:
```sql
ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST;
```

2. **Monitor replication lag** regularly:
```sql
SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
```

3. **Start application traffic** on Singapore (primary) and verify it replicates to Mumbai.

Would you like to proceed with **testing a failover**?
