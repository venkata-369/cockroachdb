### Failback: Mumbai (Primary) → Singapore (Standby)

Based on our environment, here are the exact steps: [[Failback Steps_CockroachDB Documentation](https://www.cockroachlabs.com/docs/v25.2/failover-replication#fail-back-to-the-original-primary-cluster)]

---

### Our Environment 

| Cluster | Nodes | Role |
|---|---|---|
| Mumbai | crdb-node3 (10.10.3.10), crdb-node4 (10.10.4.10) | ✅ Current Primary |
| Singapore | crdb-node8 (10.30.2.151), crdb-node9 | ⛔ Old Primary |

---
#### Why This Error Occurred

```
ERROR: tenant "main" does not exist
```

This is because **Singapore never had a `main` virtual cluster created on it**. Singapore was the original primary running as a **non-virtualized / existing cluster** — it only has the `system` virtual cluster. [[Failback Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#fail-back-to-the-original-primary-cluster)]

---

### Verify What Singapore Has

```sql
SHOW VIRTUAL CLUSTERS;
```

You will likely see:

```
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
```

No `main` virtual cluster exists on Singapore. ✅ This confirms it.

---

### Correct Failback Approach for Your Environment

Since Singapore was the **original primary without a named virtual cluster**, the failback works differently. You need to **create** the `main` virtual cluster on Singapore via replication from Mumbai:

---

### Step 1: Enable Rangefeeds on Mumbai System Virtual Cluster

On **crdb-node3** (Mumbai):

```bash
cockroach sql \
  --url "postgresql://root@10.10.3.10:26257?options=-ccluster=system&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

```sql
SET CLUSTER SETTING kv.rangefeed.enabled = true;
```

---

### Step 2: Generate Encoded Connection String for Mumbai

On **crdb-node8** (Singapore):

```bash
cockroach encode-uri \
  replicator:repl123@10.10.3.10:26257 \
  --ca-cert /home/ubuntu/certs/ca-mumbai.crt \
  --inline
```

Copy the full output connection string and this postgresql:// while creating virtual cluster
```
replicator:repl123@10.10.3.10:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJjCCAg6gAwIBAgIRAK803m4WpwvsmLorW4fS8z0wDQYJKoZIhvcNAQELBQAw%0AKzESMBAGA1UEChMJQ29ja3JvYWNoMRUwEwYDVQQDEwxDb2Nrcm9hY2ggQ0EwHhcN%0AMjYwODA3MjMwMjQ2WhcNMzYwODE1MjMwMjQ2WjArMRIwEAYDVQQKEwlDb2Nrcm9h%0AY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEP%0AADCCAQoCggEBAMOp0OBXqyRogUh5NFIYFp5xkz4GBXfUM6wVtVrKY8hWzUesAg7v%0AzHcoZX0m4IUqdZBh%2Ft64V4OX9jHnBYgkZbLB0U658FxfOfTj8nGjFdBfNgCmWI7o%0AjXyjOr715g3FegfVfM8Jdod0seSnmWFhZmizZrRECZlfoxS9d2UpvgnIiCFJ%2F8DY%0ASwHGYo%2FxiCY5eMH7tSfPhs%2BqZ1G6vdZrEP%2BdbyYmlRNhRIniK%2FuzWnQCdNSEvd8E%0Avept33RQxnVYuHDIL11UtX%2FOfIn6UE%2BlqFtKlmeOxoeDMmVUwIWlziV2e43M7uQp%0A%2Bo9wWgfMhQ1LJnHnzFxdvDkz1uqkx1E37v8CAwEAAaNFMEMwDgYDVR0PAQH%2FBAQD%0AAgLkMBIGA1UdEwEB%2FwQIMAYBAf8CAQEwHQYDVR0OBBYEFKYQVXCZ62ib0I3Kqr2H%0Amjxgjh%2BwMA0GCSqGSIb3DQEBCwUAA4IBAQAe0oYQTb0fjO9lPxKmZDNhPkypXEkt%0Azq7Pzy5Axe0dtmPXP4x0nVfjl%2B3Y%2BRQLz%2FVK9kxcoTKHfrfluK5s9PMZuMkmThzg%0AALSH%2B9TuwQeUKy0ArgKWap8uOsxA21oywg%2BTIGXEVyTSt8K3N%2FlzKprXoUz37Ziv%0AL%2FFQOXR%2FVMpCY1Cc%2BIUVT43Ot%2BOM74zuYXafA4QBCS%2BLEXfKOdP%2F4qiHqlizlbma%0AfMb3skoOzeeeFQx8jsnZDTdN3aDG50%2BKVWM430uAKfkEfKizrNqWW8WmDqqSMy28%0AGxg0MCjwTTZBleeNX42pC8V80MXEhOL%2FFNEyemx7YGv185JmM0WJAuup%0A-----END+CERTIFICATE-----%0A
```
---

### Step 3: Create `main` Virtual Cluster on Singapore via Replication

On **crdb-node8** (Singapore) system virtual cluster:
Connect to Singapore system virtual cluster:

```
cockroach sql \
  --url "postgresql://root@10.30.2.151:26257?options=-ccluster=system&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```
```sql
CREATE VIRTUAL CLUSTER main
  FROM REPLICATION OF main
  ON 'postgresql://replicator:repl123@10.10.3.10:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJjCCAg6gAwIBAgIRAK803m4WpwvsmLorW4fS8z0wDQYJKoZIhvcNAQELBQAw%0AKzESMBAGA1UEChMJQ29ja3JvYWNoMRUwEwYDVQQDEwxDb2Nrcm9hY2ggQ0EwHhcN%0AMjYwODA3MjMwMjQ2WhcNMzYwODE1MjMwMjQ2WjArMRIwEAYDVQQKEwlDb2Nrcm9h%0AY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEP%0AADCCAQoCggEBAMOp0OBXqyRogUh5NFIYFp5xkz4GBXfUM6wVtVrKY8hWzUesAg7v%0AzHcoZX0m4IUqdZBh%2Ft64V4OX9jHnBYgkZbLB0U658FxfOfTj8nGjFdBfNgCmWI7o%0AjXyjOr715g3FegfVfM8Jdod0seSnmWFhZmizZrRECZlfoxS9d2UpvgnIiCFJ%2F8DY%0ASwHGYo%2FxiCY5eMH7tSfPhs%2BqZ1G6vdZrEP%2BdbyYmlRNhRIniK%2FuzWnQCdNSEvd8E%0Avept33RQxnVYuHDIL11UtX%2FOfIn6UE%2BlqFtKlmeOxoeDMmVUwIWlziV2e43M7uQp%0A%2Bo9wWgfMhQ1LJnHnzFxdvDkz1uqkx1E37v8CAwEAAaNFMEMwDgYDVR0PAQH%2FBAQD%0AAgLkMBIGA1UdEwEB%2FwQIMAYBAf8CAQEwHQYDVR0OBBYEFKYQVXCZ62ib0I3Kqr2H%0Amjxgjh%2BwMA0GCSqGSIb3DQEBCwUAA4IBAQAe0oYQTb0fjO9lPxKmZDNhPkypXEkt%0Azq7Pzy5Axe0dtmPXP4x0nVfjl%2B3Y%2BRQLz%2FVK9kxcoTKHfrfluK5s9PMZuMkmThzg%0AALSH%2B9TuwQeUKy0ArgKWap8uOsxA21oywg%2BTIGXEVyTSt8K3N%2FlzKprXoUz37Ziv%0AL%2FFQOXR%2FVMpCY1Cc%2BIUVT43Ot%2BOM74zuYXafA4QBCS%2BLEXfKOdP%2F4qiHqlizlbma%0AfMb3skoOzeeeFQx8jsnZDTdN3aDG50%2BKVWM430uAKfkEfKizrNqWW8WmDqqSMy28%0AGxg0MCjwTTZBleeNX42pC8V80MXEhOL%2FFNEyemx7YGv185JmM0WJAuup%0A-----END+CERTIFICATE-----%0A';
```
WorkLog Output 
```
root@10.30.2.151:26257/defaultdb> CREATE VIRTUAL CLUSTER main
                               ->   FROM REPLICATION OF main
                               ->   ON
                               -> 'postgresql://replicator:repl123@10.10.3.10:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-fu
                               -> ll&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJjCCAg6gAwIBAgIRAK803m4WpwvsmLorW4fS8z0wDQYJKoZIhvcNAQELBQAw
                               -> %0AKzESMBAGA1UEChMJQ29ja3JvYWNoMRUwEwYDVQQDEwxDb2Nrcm9hY2ggQ0EwHhcN%0AMjYwODA3MjMwMjQ2WhcNMzYwODE1MjMwMjQ2WjA
                               -> rMRIwEAYDVQQKEwlDb2Nrcm9h%0AY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEP%0AADCCAQoCggEBAM
                               -> Op0OBXqyRogUh5NFIYFp5xkz4GBXfUM6wVtVrKY8hWzUesAg7v%0AzHcoZX0m4IUqdZBh%2Ft64V4OX9jHnBYgkZbLB0U658FxfOfTj8nGjFd
                               -> BfNgCmWI7o%0AjXyjOr715g3FegfVfM8Jdod0seSnmWFhZmizZrRECZlfoxS9d2UpvgnIiCFJ%2F8DY%0ASwHGYo%2FxiCY5eMH7tSfPhs%2B
                               -> qZ1G6vdZrEP%2BdbyYmlRNhRIniK%2FuzWnQCdNSEvd8E%0Avept33RQxnVYuHDIL11UtX%2FOfIn6UE%2BlqFtKlmeOxoeDMmVUwIWlziV2e
                               -> 43M7uQp%0A%2Bo9wWgfMhQ1LJnHnzFxdvDkz1uqkx1E37v8CAwEAAaNFMEMwDgYDVR0PAQH%2FBAQD%0AAgLkMBIGA1UdEwEB%2FwQIMAYBAf
                               -> 8CAQEwHQYDVR0OBBYEFKYQVXCZ62ib0I3Kqr2H%0Amjxgjh%2BwMA0GCSqGSIb3DQEBCwUAA4IBAQAe0oYQTb0fjO9lPxKmZDNhPkypXEkt%0
                               -> Azq7Pzy5Axe0dtmPXP4x0nVfjl%2B3Y%2BRQLz%2FVK9kxcoTKHfrfluK5s9PMZuMkmThzg%0AALSH%2B9TuwQeUKy0ArgKWap8uOsxA21oyw
                               -> g%2BTIGXEVyTSt8K3N%2FlzKprXoUz37Ziv%0AL%2FFQOXR%2FVMpCY1Cc%2BIUVT43Ot%2BOM74zuYXafA4QBCS%2BLEXfKOdP%2F4qiHqli
                               -> zlbma%0AfMb3skoOzeeeFQx8jsnZDTdN3aDG50%2BKVWM430uAKfkEfKizrNqWW8WmDqqSMy28%0AGxg0MCjwTTZBleeNX42pC8V80MXEhOL%
                               -> 2FFNEyemx7YGv185JmM0WJAuup%0A-----END+CERTIFICATE-----%0A';
CREATE VIRTUAL CLUSTER FROM REPLICATION 0

Time: 625ms total (execution 625ms / network 0ms)
```
---

### Step 4: Monitor Replication on Singapore

```sql
SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
```

Wait until:

```
status = replicating
replication_lag = few seconds
```

---

### Step 5: Complete Failback on Singapore

```sql
ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST;
```

---

### Step 6: Bring Singapore Online

```sql
ALTER VIRTUAL CLUSTER main START SERVICE SHARED;
```

---

### Step 7: Verify Singapore is Primary Again

```sql
SHOW VIRTUAL CLUSTERS;
```

Expected:
```
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
   3 | main   | ready      | shared
```

Set Singapore as Default
```
SET CLUSTER SETTING server.controller.default_target_cluster = 'main';
```
---
### Failback Flow Summary

```
Mumbai (Current Primary)          Singapore (Becoming Primary)
        |                                   |
        |--- PCR Replication -------------->|
        |    (Mumbai → Singapore)           |
        |                                   |
        |              COMPLETE REPLICATION TO LATEST
        |                                   |
        X (becomes standby)               ✅ Primary again
```

[[Failback Steps_CockroachDB Documents](https://www.cockroachlabs.com/docs/v25.2/failover-replication#fail-back-to-the-original-primary-cluster)]

---

#### Important Notes

| Item | Detail |
|---|---|
| Mumbai CA cert | Must be on Singapore nodes (already done earlier) |
| `replicator` user | Must exist on Mumbai with `SYSTEM REPLICATION` privilege |
| Failback timestamp | Always use `LATEST` to avoid data loss |
| After failback | Redirect application traffic back to Singapore |

### ✅ Failover Complete! `data_state = ready`

Singapore's `main` virtual cluster is now **`ready`**. 
```
root@10.30.2.151:26257/defaultdb> SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
  id | name | ingestion_job_id | source_tenant_name | source_cluster_uri | retained_time | replicated_time | replication_lag | failover_time | status
-----+------+------------------+--------------------+--------------------+---------------+-----------------+-----------------+---------------+---------
   3 | main |             NULL | NULL               | NULL               | NULL          | NULL            | NULL            |          NULL | ready
(1 row)

Time: 3ms total (execution 3ms / network 0ms)
```


### Step 2: Set Singapore as Default Cluster

```sql
SET CLUSTER SETTING server.controller.default_target_cluster = 'main';
```

---

### Step 3: Verify Singapore is Primary Again

```sql
SHOW VIRTUAL CLUSTERS;
```

Expected:
```
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
   3 | main   | ready      | shared
```

---

#### Step 4: Verify Data is Available on Singapore

```sql
SHOW DATABASES;
```

Expected:
```
  database_name
----------------
  defaultdb
  djs_appdb      ✅ Now replicated from Mumbai
  postgres
  system
```

Then verify your data:

```sql
SELECT * FROM djs_appdb.accounts;
```

Expected:
```
  id |   name   | balance
-----+----------+-----------
   1 | Venkat   |  5000.00
   2 | DJS Corp | 99999.99
```

---

#### Failback Summary ✅

| Cluster | Role | Status |
|---|---|---|
| Singapore (10.30.2.151) | ✅ Primary Again | `service_mode = shared` |
| Mumbai (10.10.3.10) | ⏸ Independent Standby | No longer replicating |

[[Failback Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#fail-back-to-the-original-primary-cluster)]

### ✅ Singapore is Fully Operational!



### Full Environment Summary ✅

| Cluster | Nodes | Role | Status |
|---|---|---|---|
| Singapore | crdb-node8 (10.30.2.151) | ✅ Primary | `main` service_mode = shared |
| Mumbai | crdb-node3 (10.10.3.10), crdb-node4 (10.10.4.10) | ⏸ Independent | No longer replicating |

---

### Final Summary 🎉

| Task | Status |
|---|---|
| Fixed crdb-node4 IPv6 binding issue | ✅ |
| Connected to Mumbai cluster | ✅ |
| Created `djs_appdb`, `accounts` table, inserted data on Mumbai | ✅ |
| Failover Mumbai → Singapore (failback) | ✅ |
| Singapore `main` virtual cluster running | ✅ |
| Data verified on Singapore | ✅ |

---





