**Podman on Windows** and prefer storing all container data under **E:\podman-instances**

### Directory Structure

Create the following directories:

```text
E:\
└── podman-instances
    └── cockdb
        ├── data
        ├── certs
        ├── logs
        ├── backup
        ├── scripts
        └── compose
```

Or using PowerShell:

```powershell
mkdir E:\podman-instances\cockdb
mkdir E:\podman-instances\cockdb\data
mkdir E:\podman-instances\cockdb\certs
mkdir E:\podman-instances\cockdb\logs
mkdir E:\podman-instances\cockdb\backup
mkdir E:\podman-instances\cockdb\scripts
mkdir E:\podman-instances\cockdb\compose
```

---

# Step 1: Create Network

```powershell
podman network create venkat-net
```

Verify:

```powershell
podman network ls
```

Expected:

```text
NETWORK ID      NAME
xxxxxxxxxxxx    venkat-net
```

---

# Step 2: Pull CockroachDB Image

```powershell
podman pull cockroachdb/cockroach:v25.2.2
```

Verify:

```powershell
podman images
```

---

# Step 3: Run Single Node

```powershell
podman run -d `
  --name cockdb1 `
  --hostname cockdb1 `
  --network venkat-net `
  -p 26257:26257 `
  -p 8080:8080 `
  -v E:\podman-instances\cockdb\data:/cockroach/cockroach-data `
  -v E:\podman-instances\cockdb\logs:/cockroach/logs `
  cockroachdb/cockroach:v25.2.2 `
  start-single-node `
  --insecure `
  --store=/cockroach/cockroach-data `
  --advertise-addr=cockdb1 `
  --http-addr=0.0.0.0:8080
```

---

# Step 4: Verify Container

```powershell
podman ps
```

Expected:

```text
NAME        STATUS
cockdb1     Up
```

---

# Step 5: Check Logs

```powershell
podman logs cockdb1
```

You should see messages indicating the node started successfully and is accepting SQL connections.

---

# Step 6: Access DB Console

Open your browser:

```text
http://localhost:8080
```

This opens the CockroachDB Admin UI.

---

# Step 7: Connect Using SQL Shell

```powershell
podman exec -it cockdb1 ./cockroach sql --insecure
```

You should get:

```sql
root@
defaultdb>
```

---

# Step 8: Verify Cluster

```sql
SHOW DATABASES;
```

```sql
SHOW CLUSTER SETTING version;
```

```sql
SHOW REGIONS;
```

```sql
SELECT node_id,address FROM crdb_internal.gossip_nodes;
```

---

# Step 9: Create Demo Database

```sql
CREATE DATABASE company;
```

```sql
USE company;
```

```sql
CREATE TABLE employee
(
    emp_id INT PRIMARY KEY,
    name STRING,
    city STRING,
    salary DECIMAL
);
```

Insert data:

```sql
INSERT INTO employee VALUES
(1,'Venkat','Hyderabad',75000),
(2,'Ravi','Bangalore',82000),
(3,'John','Chennai',90000);
```

Query:

```sql
SELECT * FROM employee;
```

---

# Step 10: Check Volume Persistence

Stop the container:

```powershell
podman stop cockdb1
```

Start it again:

```powershell
podman start cockdb1
```

Reconnect:

```powershell
podman exec -it cockdb1 ./cockroach sql --insecure
```

Run:

```sql
SELECT * FROM company.employee;
```

The data should still be present because it is stored in the mounted volume at:

```text
E:\podman-instances\cockdb\data
```

---

## Container Architecture

```text
                     Windows Host

E:\podman-instances\
        │
        └──────────────┐
                       │
                 data/
                 logs/
                 backup/
                 certs/
                       │
                       ▼
          +---------------------------+
          |     CockroachDB Container |
          |                           |
          |  SQL Port : 26257         |
          |  Admin UI : 8080          |
          |                           |
          | /cockroach/cockroach-data |
          | /cockroach/logs           |
          +---------------------------+
                       │
                 venkat-net
```

### Useful Podman Commands

```powershell
podman ps
podman images
podman logs cockdb1
podman inspect cockdb1
podman stats
podman stop cockdb1
podman start cockdb1
podman restart cockdb1
podman rm -f cockdb1
```

```
Remove-Item -Recurse -Force E:\podman-instances\cockdb
```

persistent **single-node CockroachDB** on the `venkat-net` network with data stored under `E:\podman-instances\cockdb`, 

---Next Session, making it easy to expand later into a 3-node cluster for your demo.

--- podman pull cockroachdb/cockroach:v24.3.16
