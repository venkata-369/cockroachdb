#### ✅ Failover Complete! Mumbai is Now the Primary Cluster!

Every step succeeded perfectly: [[Failover Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#step-2-complete-the-failover)]

| Step | Command | Result |
|---|---|---|
| Initiate failover | `ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST` | ✅ `failover_time` set |
| Monitor | `SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS` | ✅ `replication pending failover` |
| Start service | `ALTER VIRTUAL CLUSTER main START SERVICE SHARED` | ✅ Done |
| Verify | `SHOW VIRTUAL CLUSTERS` | ✅ `main` is `ready` + `shared` |

---

## ⏭️ Step 6: Make Mumbai the Default Cluster

Run this on **crdb-node3** (Mumbai): [[Failover Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#step-2-complete-the-failover)]

```sql
SET CLUSTER SETTING server.controller.default_target_cluster='main';
```

---

## ✅ Final PCR Failover Summary

| Item | Status |
|---|---|
| Singapore (old primary) | ⛔ No longer primary |
| Mumbai (new primary) | ✅ Now serving as primary |
| `main` virtual cluster | ✅ `ready` + `shared` |
| Replication lag at failover | ~7 seconds |
| Data loss | Minimal — used `LATEST` |

---

#### ⚠️ Important Notes

- Singapore and Mumbai are now **completely independent** clusters
- You need to **redirect application traffic** to Mumbai using your load balancer or DNS
- To set up **replication back** from Mumbai → Singapore (failback), you can run:

```sql
ALTER VIRTUAL CLUSTER main START REPLICATION OF system
  ON 'postgresql://replicator:repl123@10.10.3.10:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=...';
```

[[Failback Steps CockroachDB Documentation](https://www.cockroachlabs.com/docs/v25.2/failover-replication#failback)]

---

Would you like to test **failback** from Mumbai back to Singapore, or set up **application connectivity** to the new primary?
