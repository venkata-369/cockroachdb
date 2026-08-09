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

## Failover: Singapore → Mumbai

The `ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST` command must be run on the **Mumbai (standby) cluster** — NOT Singapore. [[Failover Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#failover)]

---

## Step 1: Connect to Mumbai SQL Shell

On **crdb-node3** (Mumbai):

```bash
cockroach sql \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10
```

---

## Step 2: Initiate Failover on Mumbai

```sql
ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST;
```

Expected output:
```
        failover_time
----------------------------------
1695922878030920020.0000000000
(1 row)
```

[[Failover Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#failover)]

---

## Step 3: Monitor Failover Status

```sql
SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
```

Watch for `status` to change:

| Status | Meaning |
|---|---|
| `replicating` | Still replicating |
| `replication pending failover` | Failover in progress |
| `ready` | ✅ Failover complete |

---

## Step 4: Start Service on Mumbai After Failover

Once status shows `ready`, run: [[ALTER VIRTUAL CLUSTER](https://www.cockroachlabs.com/docs/v25.2/alter-virtual-cluster#examples)]

```sql
ALTER VIRTUAL CLUSTER main START SERVICE SHARED;
```

---

## Step 5: Verify Mumbai is Now Primary

```sql
SHOW VIRTUAL CLUSTERS;
```

Expected:
```
  id |  name  | data_state | service_mode
-----+--------+------------+--------------
   1 | system | ready      | shared
   3 | main   | ready      | shared
```

---

## ⚠️ Key Points

| Item | Detail |
|---|---|
| Run failover command | **Mumbai (standby)** only |
| Singapore | Will stop being primary after failover |
| `LATEST` | Minimizes data loss — uses most recent replicated timestamp |
| After failover | Mumbai becomes the new **primary** |

WorkLog Output:- 
```
root@10.10.3.10:26257/defaultdb> SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
  id | name |  ingestion_job_id   | source_tenant_name |                     source_cluster_uri                      |        retained_time        |    replicated_time     | replication_lag | failover_time |   status
-----+------+---------------------+--------------------+-------------------------------------------------------------+-----------------------------+------------------------+-----------------+---------------+--------------
   3 | main | 1199898364513452033 | system             | postgresql://replicator:redacted@10.30.2.151:26257?redacted | 2026-08-09 04:38:41.3816+00 | 2026-08-09 04:52:45+00 | 00:00:07.57706  |          NULL | replicating
(1 row)

Time: 9ms total (execution 8ms / network 0ms)

root@10.10.3.10:26257/defaultdb> ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST;
          failover_time
----------------------------------
  1786251210000000000.0000000000
(1 row)

Time: 28ms total (execution 28ms / network 0ms)

root@10.10.3.10:26257/defaultdb> SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
  id | name |  ingestion_job_id   | source_tenant_name |                     source_cluster_uri                      |        retained_time        |    replicated_time     | replication_lag |         failover_time          |            status
-----+------+---------------------+--------------------+-------------------------------------------------------------+-----------------------------+------------------------+-----------------+--------------------------------+-------------------------------
   3 | main | 1199898364513452033 | system             | postgresql://replicator:redacted@10.30.2.151:26257?redacted | 2026-08-09 04:38:41.3816+00 | 2026-08-09 04:53:30+00 | 00:00:19.714801 | 1786251210000000000.0000000000 | replication pending failover
(1 row)

Time: 7ms total (execution 7ms / network 0ms)

root@10.10.3.10:26257/defaultdb> ALTER VIRTUAL CLUSTER main START SERVICE SHARED;
ALTER VIRTUAL CLUSTER SERVICE

Time: 10ms total (execution 10ms / network 1ms)

root@10.10.3.10:26257/defaultdb> SHOW VIRTUAL CLUSTERS;
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
   3 | main   | ready      | shared
(2 rows)

Time: 4ms total (execution 4ms / network 0ms)
---From Singapore Node
root@10.30.2.151:26257/defaultdb> SHOW VIRTUAL CLUSTERS;
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
(1 row)

Time: 5ms total (execution 5ms / network 0ms)
```

