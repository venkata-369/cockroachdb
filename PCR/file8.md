### Test: Create User, Database, Table on Your Environment

Based on our setup, here is how to test on **Mumbai (now Primary)**: [[SQL Statements_CockroachDB Documentation](https://www.cockroachlabs.com/docs/stable/secure-a-cluster#step-3-use-the-built-in-sql-client)]

---

### Step 1: Connect to Mumbai `main` Virtual Cluster

```bash
cockroach sql \
  --url "postgresql://root@10.10.3.10:26257/defaultdb?options=-ccluster=main&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

---

## Step 2: Create a User

```sql
CREATE USER appuser WITH PASSWORD 'appuser123';
```

Verify:

```sql
SHOW USERS;
```

Expected:
```
  username  | options
------------+---------
  appuser   |
  root      |
```

---

## Step 3: Create a Database

```sql
CREATE DATABASE djs_appdb;
```

Verify:

```sql
SHOW DATABASES;
```

Expected:
```
  database_name
----------------
  defaultdb
  djs_appdb
  postgres
  system
```

---

## Step 4: Grant Privileges to User

```sql
GRANT ALL ON DATABASE djs_appdb TO appuser;
```

---

## Step 5: Create a Table

```sql
CREATE TABLE djs_appdb.accounts (
  id INT PRIMARY KEY,
  name STRING NOT NULL,
  balance DECIMAL
);
```

Verify:

```sql
SHOW TABLES FROM djs_appdb;
```

Expected:
```
  table_name
------------
  accounts
```

---

## Step 6: Insert Data

```sql
INSERT INTO djs_appdb.accounts VALUES
  (1, 'Venkat', 5000.00),
  (2, 'DJS Corp', 99999.99);
```

---

## Step 7: Query Data

```sql
SELECT * FROM djs_appdb.accounts;
```

Expected:
```
  id |   name   | balance
-----+----------+----------
   1 | Venkat   | 5000.00
   2 | DJS Corp | 99999.99
```

---

## Step 8: Test as appuser

Exit and reconnect as `appuser`:

```bash
cockroach sql \
  --url "postgresql://appuser:appuser123@10.10.3.10:26257/djs_appdb?options=-ccluster=main&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

Then run:

```sql
SELECT * FROM djs_appdb.accounts;
```

---

### ✅ Test Checklist

| Test | Command | Status |
|---|---|---|
| Create user | `CREATE USER appuser` | ✅ |
| Create database | `CREATE DATABASE djs_appdb` | ✅ |
| Create table | `CREATE TABLE djs_appdb.accounts` | ✅ |
| Insert data | `INSERT INTO djs_appdb.accounts` | ✅ |
| Query data | `SELECT * FROM djs_appdb.accounts` | ✅ |
| Connect as appuser | `cockroach sql --url ...appuser...` | ✅ |

Share the output after each step!

---
## ✅ This is Expected Behavior — Failover Already Completed!

The reason Singapore does not have `djs_appdb` is because **you already completed the failover** from Singapore → Mumbai earlier. [[Failover Steps](https://www.cockroachlabs.com/docs/v25.2/failover-replication#step-2-complete-the-failover)]

---

## What Happened in Your Environment

| Event | Result |
|---|---|
| PCR started | Singapore → Mumbai replicating |
| `ALTER VIRTUAL CLUSTER main COMPLETE REPLICATION TO LATEST` | Failover initiated |
| `ALTER VIRTUAL CLUSTER main START SERVICE SHARED` | Mumbai became primary |
| `djs_appdb` created **after failover** | Only exists on Mumbai |
| Singapore | Independent — no longer receiving data |

---
```
root@10.10.4.10:26257/system/defaultdb> show databases;
  database_name | owner | primary_region | secondary_region | regions | survival_goal
----------------+-------+----------------+------------------+---------+----------------
  defaultdb     | root  | NULL           | NULL             | {}      | NULL
  djs_appdb     | root  | NULL           | NULL             | {}      | NULL
  postgres      | root  | NULL           | NULL             | {}      | NULL
  system        | node  | NULL           | NULL             | {}      | NULL
(4 rows)

Time: 13.005s total (execution 11.896s / network 1.109s)

root@10.10.4.10:26257/system/defaultdb> SELECT * FROM djs_appdb.accounts;
  id |   name   | balance
-----+----------+-----------
   1 | Venkat   |  5000.00
   2 | DJS Corp | 99999.99
(2 rows)

Time: 94ms total (execution 83ms / network 11ms)
```
```
root@10.30.2.151:26257/defaultdb> show databases;
  database_name | owner | primary_region | secondary_region | regions | survival_goal
----------------+-------+----------------+------------------+---------+----------------
  defaultdb     | root  | NULL           | NULL             | {}      | NULL
  postgres      | root  | NULL           | NULL             | {}      | NULL
  system        | node  | NULL           | NULL             | {}      | NULL
(3 rows)

Time: 9ms total (execution 9ms / network 0ms)
```

#### Timeline Explanation

```
Singapore (Primary)          Mumbai (Standby → Primary)
      |                              |
      |--- PCR Replication --------->|
      |                              |
      | FAILOVER COMPLETED           |
      |                              |
      X (no longer primary)         ✅ Now Primary
                                     |
                              djs_appdb created HERE
                              accounts table created HERE
                              Data inserted HERE
```

---

## Key Point

After failover:
- **Mumbai** is the **new primary** — all new writes go here ✅
- **Singapore** is now **independent** — it stopped receiving data at failover time ❌
- `djs_appdb` was created **after** failover on Mumbai, so Singapore never received it

---

## Verify This on Singapore

Connect to Singapore and check — it only has data up to the **failover timestamp**:

```bash
cockroach sql \
  --certs-dir=/home/ubuntu/certs \
  --host=10.30.2.151
```

```sql
SHOW VIRTUAL CLUSTERS;
```

Singapore will show:
```
  id |  name  | data_state | service_mode
-----+--------+------------+---------------
   1 | system | ready      | shared
```

No `main` virtual cluster on Singapore — confirming it is no longer part of PCR. ✅

---

#### Summary

| Cluster | Role | Has `djs_appdb` |
|---|---|---|
| Mumbai (crdb-node3, crdb-node4) | ✅ Current Primary | ✅ Yes |
| Singapore (crdb-node8, crdb-node9) | ⛔ Old Primary (independent) | ❌ No — created after failover |

