## CockroachDB v25.2 Backup and Restore Labs

### Lab Environment

### Cluster Topology

```
crdb-node1   10.10.1.10
crdb-node2   10.10.1.11
crdb-node3   10.10.1.12
```

Cluster started using:

```bash
cockroach start \
  --insecure \
  --advertise-addr=<node-ip> \
  --listen-addr=<node-ip>:26257 \
  --http-addr=<node-ip>:8080 \
  --join=10.10.1.10:26257,10.10.1.11:26257,10.10.1.12:26257
```

Connect to SQL:

```bash
cockroach sql --insecure --host=10.10.1.10:26257
```

Verify the cluster:

```sql
SHOW DATABASES;
SHOW CLUSTER SETTING version;
SHOW REGIONS FROM CLUSTER;
```

Expected version:

```
25.2
```

---

### Lab 1 – Full Database Backup

### Step 1 – Verify the databases

```sql
SHOW DATABASES;
```

Expected output:

```
blrdb
company
defaultdb
postgres
system
train
```

---

### Step 2 – Verify demo tables

```sql
USE defaultdb;

SHOW TABLES;
```

Expected:

```
customers
orders
employees
```

---

### Step 3 – Take a full backup

```sql
BACKUP DATABASE defaultdb INTO 'nodelocal://1/full_backup';
```

---

### Step 4 – Monitor backup

```sql
SHOW JOBS;
```

Expected:

```
Job Type : BACKUP
Status   : succeeded
```

---

### Step 5 – List backup collections (v25.2)

```sql
SHOW BACKUPS IN 'nodelocal://1/full_backup';
```

Example:

```
/2026/07/21-000432.63
```

---

### Step 6 – Inspect the backup

```sql
SHOW BACKUP FROM '/2026/07/21-000432.63' IN 'nodelocal://1/full_backup';
```

---

### Step 7 – View schemas

```sql
SHOW BACKUP SCHEMAS FROM '/2026/07/21-000432.63' IN 'nodelocal://1/full_backup';
```

---

### Step 8 – View backup ranges

```sql
SHOW BACKUP RANGES FROM '/2026/07/21-000432.63' IN 'nodelocal://1/full_backup';
```

---

# Lab 2 – Incremental Backup

Create an initial backup:

```sql
BACKUP DATABASE defaultdb INTO 'nodelocal://1/inc_demo';
```

Insert sample data:

```sql
INSERT INTO customers VALUES (100001,'Backup User','backup@test.com');
```

```sql
INSERT INTO orders
VALUES
(9001,100001,'Laptop',1,65000);
```

Verify:

```sql
SELECT *
FROM customers
WHERE customer_id=100001;
```

Create an incremental backup:

```sql
BACKUP DATABASE defaultdb
INTO LATEST IN 'nodelocal://1/inc_demo';
```

View backup chain:

```sql
SHOW BACKUPS IN 'nodelocal://1/inc_demo';
```

Inspect the latest backup:

```sql
SHOW BACKUP
FROM LATEST
IN 'nodelocal://1/inc_demo';
```

---

# Lab 3 – Database Restore

Verify row counts before restore:

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM employees;
```

Drop the database:

```sql
DROP DATABASE defaultdb CASCADE;
```

Restore:

```sql
RESTORE DATABASE defaultdb
FROM LATEST
IN 'nodelocal://1/inc_demo';
```

Verify:

```sql
SHOW TABLES;
```

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM employees;
```

---

# Lab 4 – Table-Level Backup and Restore

Backup a single table:

```sql
BACKUP TABLE defaultdb.public.customers
INTO 'nodelocal://1/customer_backup';
```

View backup:

```sql
SHOW BACKUPS IN 'nodelocal://1/customer_backup';
```

Restore:

```sql
RESTORE TABLE defaultdb.public.customers
FROM LATEST
IN 'nodelocal://1/customer_backup';
```

---

# Lab 5 – Scheduled Backups

Create a schedule:

```sql
CREATE SCHEDULE daily_backup
FOR BACKUP DATABASE defaultdb
INTO 'nodelocal://1/scheduled'
RECURRING '@daily';
```
```
CREATE SCHEDULE daily_backup_8pm
FOR BACKUP DATABASE defaultdb
INTO 'nodelocal://1/scheduled'
RECURRING '0 20 * * *'
WITH SCHEDULE OPTIONS timezone='Asia/Kolkata';
```


View schedules:

```sql
SHOW SCHEDULES;
```

Pause:

```sql
PAUSE SCHEDULE <schedule_id>;
```

Resume:

```sql
RESUME SCHEDULE <schedule_id>;
```

Drop:

```sql
DROP SCHEDULE <schedule_id>;
```

---

# Lab 6 – Encrypted Backup

Create an encrypted backup:

```sql
BACKUP DATABASE defaultdb
INTO 'nodelocal://1/encrypted_backup'
WITH encryption_passphrase='MyStrongPassword123';
```

Restore:

```sql
RESTORE DATABASE defaultdb
FROM LATEST
IN 'nodelocal://1/encrypted_backup'
WITH encryption_passphrase='MyStrongPassword123';
```

---

# Lab 7 – Backup to Amazon S3

Backup:

```sql
BACKUP DATABASE defaultdb
INTO 's3://mybucket/crdb-backups?AWS_ACCESS_KEY_ID=<ACCESS_KEY>&AWS_SECRET_ACCESS_KEY=<SECRET_KEY>&AWS_REGION=ap-south-1';
```

Verify:

```sql
SHOW BACKUPS
IN 's3://mybucket/crdb-backups?AWS_ACCESS_KEY_ID=<ACCESS_KEY>&AWS_SECRET_ACCESS_KEY=<SECRET_KEY>&AWS_REGION=ap-south-1';
```

Restore:

```sql
RESTORE DATABASE defaultdb
FROM LATEST
IN 's3://mybucket/crdb-backups?AWS_ACCESS_KEY_ID=<ACCESS_KEY>&AWS_SECRET_ACCESS_KEY=<SECRET_KEY>&AWS_REGION=ap-south-1';
```

---

# Lab 8 – Point-in-Time Recovery (PITR)

View current cluster time:

```sql
SELECT now();
```

Insert new data:

```sql
INSERT INTO customers
VALUES
(100010,'Recovery User','recover@test.com');
```

Restore the database to a previous timestamp:

```sql
RESTORE DATABASE defaultdb
FROM LATEST
IN 'nodelocal://1/inc_demo'
AS OF SYSTEM TIME '-5m';
```

Verify whether the new row exists:

```sql
SELECT *
FROM customers
WHERE customer_id=100010;
```

---

# Lab 9 – Backup Validation

Display backup history:

```sql
SHOW BACKUPS IN 'nodelocal://1/full_backup';
```

Display backup details:

```sql
SHOW BACKUP
FROM LATEST
IN 'nodelocal://1/full_backup';
```

Display schemas:

```sql
SHOW BACKUP SCHEMAS
FROM LATEST
IN 'nodelocal://1/full_backup';
```

Display ranges:

```sql
SHOW BACKUP RANGES
FROM LATEST
IN 'nodelocal://1/full_backup';
```

View backup jobs:

```sql
SHOW JOBS;
```

View schedules:

```sql
SHOW SCHEDULES;
```

Verify data:

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM employees;
```

---

# Lab 10 – Backup Job Monitoring

List active jobs:

```sql
SHOW JOBS;
```

Show currently running SQL statements:

```sql
SHOW CLUSTER QUERIES;
```

View node status:

```sql
SHOW CLUSTER SESSIONS;
```

Open the DB Console:

```
http://10.10.1.10:8080
```

Navigate to:

* Jobs
* Metrics
* Databases
* Node Status

---

# Lab 11 – Disaster Recovery Simulation

1. Take a full backup.
2. Stop one node:

```bash
sudo systemctl stop cockroach
```

3. Verify cluster health:

```sql
SHOW RANGES FROM DATABASE defaultdb;
```

4. Restart the node:

```bash
sudo systemctl start cockroach
```

5. Restore the database from the latest backup:

```sql
RESTORE DATABASE defaultdb
FROM LATEST
IN 'nodelocal://1/full_backup';
```

6. Verify data integrity:

```sql
SELECT COUNT(*) FROM customers;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM employees;
```

## Notes for CockroachDB v25.2

* Do **not** create Linux directories such as `/backups/full` for `nodelocal://` backups. CockroachDB manages the node-local backup location through its configured external I/O directory.
* Use `SHOW BACKUPS IN '<collection>'` to list backups in a collection.
* Use `SHOW BACKUP FROM LATEST IN '<collection>'` or `SHOW BACKUP FROM '<backup_path>' IN '<collection>'` to inspect a specific backup.
* Verify your CockroachDB edition before using enterprise features such as scheduled backups, cloud storage backups, and encrypted backups, as their availability depends on your license.
