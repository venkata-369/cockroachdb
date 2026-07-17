### On 3 Node Cluster, we will have a hands-on demo
| Node  | Private IP     | Role        |
| ----- | -------------- | ----------- |
| Node1 | **10.10.1.10** | CockroachDB |
| Node2 | **10.10.2.10** | CockroachDB |
| Node3 | **10.10.3.10** | CockroachDB |

You are using **--insecure** mode.

This is a perfect lab to understand CockroachDB internals. Below is a complete topic-wise hands-on lab with commands, expected output, and explanations.

---

### Lab 1 - Cluster Verification

Connect from Node1

```bash
ssh ubuntu@10.10.1.10

cockroach sql --host=10.10.1.10 --insecure
```

Check cluster

```sql
SHOW CLUSTER SETTING version;

SHOW NODES;

SHOW REGIONS FROM CLUSTER;
```

Expected

```
Node1
Node2
Node3
```

---

### Lab 2 - Create Database

```sql
CREATE DATABASE company;

USE company;
```

Verify

```sql
SHOW DATABASES;
```

---

### Lab 3 - Create Tables

Employee

```sql
CREATE TABLE employees
(
emp_id INT PRIMARY KEY,
first_name STRING,
last_name STRING,
department STRING,
salary DECIMAL,
city STRING,
created_at TIMESTAMP DEFAULT now()
);
```

Orders

```sql
CREATE TABLE orders
(
order_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
emp_id INT,
amount DECIMAL,
status STRING,
created_at TIMESTAMP DEFAULT now()
);
```

Customers

```sql
CREATE TABLE customers
(
customer_id INT PRIMARY KEY,
customer_name STRING,
city STRING
);
```

---

#### Lab 4 - Load 1 Million Rows

Employees

```sql
INSERT INTO employees
SELECT
g,
'EMP'||g,
'LAST'||g,
CASE
WHEN g%5=0 THEN 'HR'
WHEN g%5=1 THEN 'IT'
WHEN g%5=2 THEN 'Finance'
WHEN g%5=3 THEN 'Sales'
ELSE 'Support'
END,
(random()*90000)+10000,
CASE
WHEN g%4=0 THEN 'Hyderabad'
WHEN g%4=1 THEN 'Bangalore'
WHEN g%4=2 THEN 'Chennai'
ELSE 'Pune'
END,
now()
FROM generate_series(1,1000000) g;
```

Orders

```sql
INSERT INTO orders
SELECT
gen_random_uuid(),
(random()*1000000)::INT,
(random()*50000),
'Completed',
now()
FROM generate_series(1,1000000);
```

Customers

```sql
INSERT INTO customers
SELECT
g,
'Customer'||g,
'Hyderabad'
FROM generate_series(1,1000000) g;
```

Verify

```sql
SELECT count(*) FROM employees;

SELECT count(*) FROM orders;

SELECT count(*) FROM customers;
```

---

#### DistSQL Demo

CockroachDB automatically distributes query execution.

Enable plan

```sql
EXPLAIN ANALYZE
SELECT department,
AVG(salary)
FROM employees
GROUP BY department;
```

Observe

```
Distribution: Full

Vectorized: true
```

Meaning

```
Node1 scans data

↓

Node2 scans another portion

↓

Node3 scans another portion

↓

Results merged
```

Architecture

```
                Client

                  |

              SQL Gateway
             (10.10.1.10)

          DistSQL Planner

        /        |        \

10.10.1.10 10.10.2.10 10.10.3.10

 Scan         Scan        Scan

        Aggregate Merge
```

Interview Question

How do you know DistSQL is used?

Answer

```
EXPLAIN ANALYZE
```

Look for

```
Distribution: Full
```

---

### Ranges Demo

Every table is divided into ranges.

Check

```sql
SHOW RANGES FROM TABLE employees;
```

Output

```
Range ID

Start Key

End Key

Replicas

Leaseholder
```

Example

```
Range 100

Replicas

Node1

Node2

Node3
```

Large tables create many ranges automatically.

Check number of ranges

```sql
SELECT count(*) FROM [SHOW RANGES FROM TABLE employees];
```

---

### Range Splits Demo

Create a split

```sql
ALTER TABLE employees SPLIT AT VALUES (250000),(500000),(750000);
```

Verify

```sql
SHOW RANGES FROM TABLE employees;
```

Observe

```
Range1

1-249999

Range2

250000-499999

Range3

500000-749999

Range4

750000-1000000
```

Now execute

```sql
SELECT * FROM employees WHERE emp_id=600000;
```

Only one range is scanned.

---

### Range Merge Demo

Remove manual split

```sql
ALTER TABLE employees UNSPLIT AT VALUES (250000);

ALTER TABLE employees UNSPLIT AT VALUES (500000);

ALTER TABLE employees UNSPLIT AT VALUES (750000);
```

Automatic merge occurs later.

Verify

```sql
SHOW RANGES FROM TABLE employees;
```

---

### Replicas Demo

Check replicas

```sql
SHOW RANGES FROM TABLE employees;
```

Example

```
Range 101

Replicas

1

2

3
```

Meaning

```
Replica

↓

10.10.1.10

Replica

↓

10.10.2.10

Replica

↓

10.10.3.10
```

Each replica contains identical data.

---

### Leaseholder Demo

Find leaseholder

```sql
SHOW RANGES FROM TABLE employees;
```

Example

```
Lease Holder

2
```

Meaning

```
Leaseholder

↓

10.10.2.10
```

Stop Node2

```bash
sudo systemctl stop cockroach
```

Run again

```sql
SHOW RANGES FROM TABLE employees;
```

Leaseholder changes

```
Before

10.10.2.10

↓

After

10.10.1.10
```

No manual intervention required.

---

### Gossip Protocol Demo

Run

```sql
SHOW NODES;
```

Every node knows

```
Node1

↓

Node2

↓

Node3
```

Now stop Node3

```bash
sudo systemctl stop cockroach
```

Run

```sql
SHOW NODES;
```

Node3

```
Unavailable
```

Start again

```bash
sudo systemctl start cockroach
```

After a few seconds

```
Node3

Healthy
```

The node automatically learns cluster state through CockroachDB's internal networking and node liveness mechanisms (historically called the Gossip subsystem).

---

### Raft Consensus Demo

Current replicas

```
10.10.1.10

10.10.2.10

10.10.3.10
```

Insert

```sql
INSERT INTO employees
VALUES
(
2000001,
'VENKAT',
'SARATH',
'IT',
50000,
'Hyderabad',
now()
);
```

Flow

```
Client

↓

Leaseholder

↓

Raft

↓

10.10.1.10

↓

10.10.2.10

↓

10.10.3.10

↓

Majority

↓

Commit
```

Stop Node3

```bash
sudo systemctl stop cockroach
```

Insert

```sql
INSERT INTO employees
VALUES
(
2000002,
'TEST',
'USER',
'IT',
60000,
'Bangalore',
now()
);
```

Still succeeds because

```
Majority = 2

Node1

+

Node2
```

Stop Node2 also

```
Only Node1 alive
```

Insert

```sql
INSERT INTO employees
VALUES
(
2000003,
'FAIL',
'USER',
'IT',
100,
'Pune',
now()
);
```

Fails because the Raft quorum (2 of 3 replicas) is no longer available.

---

### Metadata Tables Demo

List system tables

```sql
USE system;

SHOW TABLES;
```

Useful metadata tables

```sql
SELECT * FROM system.namespace LIMIT 10;
```

Database and object namespace.

```sql
SELECT * FROM system.descriptor LIMIT 10;
```

Descriptors for databases, tables, indexes, etc.

```sql
SELECT * FROM system.jobs LIMIT 20;
```

Background jobs (backups, restores, schema changes).

```sql
SELECT * FROM system.users;
```

Users.

```sql
SELECT * FROM system.role_members;
```

Role memberships.

```sql
SELECT * FROM system.settings;
```

Cluster settings.

```sql
SELECT * FROM system.locations;
```

Location metadata (if configured).

```sql
SELECT * FROM system.comments;
```

Comments on schema objects.

---

#### Complete Write Path

```
Application

      │

      ▼

SQL Gateway (10.10.1.10)

      │

      ▼

Leaseholder (10.10.2.10)

      │

      ▼

Raft Consensus

      │

 ┌────┴───────────┐

 ▼                ▼

Replica      Replica

10.10.1.10   10.10.3.10

      │

 Majority ACK (2/3)

      │

      ▼

 Commit

      │

      ▼

 Client receives SUCCESS
```

