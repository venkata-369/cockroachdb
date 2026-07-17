### CockroachDB Internals Lab Summary

| Node   | Hostname   | Private IP     |
| ------ | ---------- | -------------- |
| Node-1 | crdb-node1 | **10.10.1.10** |
| Node-2 | crdb-node2 | **10.10.2.10** |
| Node-3 | crdb-node3 | **10.10.3.10** |

Cluster Mode:

* 3-node CockroachDB Cluster
* AWS EC2
* `--insecure`
* RF = 3 (default)
* One leaseholder per range
* Raft consensus enabled
* DistSQL enabled

---

#### CockroachDB Internals Lab Summary

The following demos will be covered.

* Create a sample database
* Create multiple tables
* Load one million records into each table
* Verify data distribution
* Understand DistSQL execution
* Understand Replicas (RF=3)
* Identify Leaseholders
* Observe Leaseholder movement during node failure
* Understand Gossip Protocol (node discovery and cluster state)
* Understand Raft Consensus (write path and quorum)
* Explore CockroachDB Metadata Tables
* Verify replication factor and replica placement
* Interview questions and expected observations

---

# Lab Database

```sql
CREATE DATABASE trainingdb;

USE trainingdb;
```

---

# Create Sample Tables

Instead of generic names like `employees` and `orders`, use business-oriented names that make the labs more realistic.

## Table 1 – Customers

```sql
CREATE TABLE customers
(
    customer_id INT PRIMARY KEY,
    customer_name STRING,
    city STRING,
    mobile STRING,
    created_date TIMESTAMP DEFAULT now()
);
```

Purpose

* Customer master data
* Primary key lookups
* Range split demonstrations

---

## Table 2 – Products

```sql
CREATE TABLE products
(
    product_id INT PRIMARY KEY,
    product_name STRING,
    category STRING,
    price DECIMAL,
    created_date TIMESTAMP DEFAULT now()
);
```

Purpose

* DistSQL
* Full table scans
* Aggregation demos

---

## Table 3 – Sales

```sql
CREATE TABLE sales
(
    sale_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id INT,
    product_id INT,
    quantity INT,
    amount DECIMAL,
    sale_date TIMESTAMP DEFAULT now()
);
```

Purpose

* Large joins
* Distributed execution
* Raft write demonstrations

---

# Load 1 Million Rows

## Customers

```sql
INSERT INTO customers
SELECT
g,
'Customer-'||g,
CASE
WHEN g%4=0 THEN 'Hyderabad'
WHEN g%4=1 THEN 'Bangalore'
WHEN g%4=2 THEN 'Chennai'
ELSE 'Pune'
END,
'90000'||lpad(g::STRING,5,'0'),
now()
FROM generate_series(1,1000000) g;
```

---

## Products

```sql
INSERT INTO products
SELECT
g,
'Product-'||g,
CASE
WHEN g%5=0 THEN 'Laptop'
WHEN g%5=1 THEN 'Mobile'
WHEN g%5=2 THEN 'TV'
WHEN g%5=3 THEN 'Printer'
ELSE 'Monitor'
END,
(random()*90000)+1000,
now()
FROM generate_series(1,1000000) g;
```

---

## Sales

```sql
INSERT INTO sales
SELECT
gen_random_uuid(),
(random()*1000000)::INT,
(random()*1000000)::INT,
(random()*20)::INT,
(random()*100000),
now()
FROM generate_series(1,1000000);
```

---

Verify

```sql
SELECT count(*) FROM customers;

SELECT count(*) FROM products;

SELECT count(*) FROM sales;
```

Expected

```
1000000

1000000

1000000
```

---

# Demo 1 — DistSQL

## Objective

Understand how CockroachDB executes SQL across multiple nodes.

Imagine you connect to

```
10.10.1.10
```

Even though you connect only to Node-1, the query executes on all three nodes.

```
                   Client

                     |

             SQL Gateway

               10.10.1.10

                     |

             DistSQL Planner

        -----------------------------

        |             |             |

10.10.1.10      10.10.2.10     10.10.3.10

 Scan            Scan            Scan

        Aggregate Results
```

Run

```sql
EXPLAIN ANALYZE
SELECT
category,
AVG(price)
FROM products
GROUP BY category;
```

Observe

```
Distribution : Full

Vectorized : true
```

Meaning

* Node1 scanned part of the data
* Node2 scanned another part
* Node3 scanned the remaining data
* Results were merged before returning to the client

Interview Question

**How do you know CockroachDB used DistSQL?**

Answer

```
EXPLAIN ANALYZE

Distribution : Full
```

---

# Demo 2 — Replication Factor (RF = 3)

CockroachDB automatically stores three replicas for every range in your cluster.

Current cluster

```
Node-1

10.10.1.10

Node-2

10.10.2.10

Node-3

10.10.3.10
```

Check ranges

```sql
SHOW RANGES FROM TABLE customers;
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
Replica 1

10.10.1.10

Replica 2

10.10.2.10

Replica 3

10.10.3.10
```

Every range has exactly three replicas because the cluster is using RF = 3.

Verify the replication setting:

```sql
SHOW ZONE CONFIGURATION FOR TABLE customers;
```

Look for:

```
num_replicas = 3
```

---

# Demo 3 — Leaseholders

A range may have three replicas, but only one leaseholder.

Check

```sql
SHOW RANGES FROM TABLE customers;
```

Example

```
Range ID      Lease Holder

100          2
```

Meaning

```
Leaseholder

↓

10.10.2.10
```

Stop Node-2

```
sudo systemctl stop cockroach
```

Run

```sql
SHOW RANGES FROM TABLE customers;
```

Now

```
Lease Holder

1
```

Meaning

```
Lease transferred

10.10.2.10

↓

10.10.1.10
```

The application continues running because another replica acquired the lease.

---

# Demo 4 — Gossip Protocol

CockroachDB nodes continuously exchange cluster metadata.

Initially

```
10.10.1.10

↓

10.10.2.10

↓

10.10.3.10
```

Verify cluster membership

```sql
SHOW NODES;
```

Stop

```
10.10.3.10
```

```
sudo systemctl stop cockroach
```

Run

```sql
SHOW NODES;
```

Expected

```
Node3

Unavailable
```

Restart

```
sudo systemctl start cockroach
```

Within a short time

```
Node3

Healthy
```

This demonstrates that the cluster automatically shares node status and updates membership information internally.

> **Note:** In modern CockroachDB versions, the original Gossip subsystem has largely been replaced by newer mechanisms (such as the KV layer and node liveness), but the term "Gossip Protocol" is still commonly used in interviews and documentation to describe cluster metadata propagation.

---

# Demo 5 — Raft Consensus

Suppose the leaseholder for a range is on:

```
10.10.2.10
```

Insert

```sql
INSERT INTO customers
VALUES
(
1000001,
'Venkat',
'Hyderabad',
'9999999999',
now()
);
```

Write path

```
Client

↓

10.10.1.10 (Gateway)

↓

Leaseholder

10.10.2.10

↓

Raft Replication

↓

10.10.1.10

10.10.2.10

10.10.3.10

↓

Majority ACK

↓

Commit
```

Now stop

```
10.10.3.10
```

Insert again

```sql
INSERT INTO customers
VALUES
(
1000002,
'Sarath',
'Bangalore',
'8888888888',
now()
);
```

Result

```
Success
```

Reason

```
Majority = 2

Node1

+

Node2
```

Now stop Node2 as well.

Only

```
10.10.1.10
```

is running.

Try another insert.

Expected

```
Transaction Failed
```

Reason

```
No Majority

Need

2

Only

1
```

This is Raft quorum in action.

---

# Demo 6 — Metadata Tables

Switch

```sql
USE system;
```

List tables

```sql
SHOW TABLES;
```

Useful metadata tables

### Namespace

```sql
SELECT * FROM system.namespace LIMIT 10;
```

Contains mappings between object names and IDs.

### Descriptor

```sql
SELECT * FROM system.descriptor LIMIT 10;
```

Stores metadata for databases, tables, indexes, and schemas.

### Jobs

```sql
SELECT * FROM system.jobs LIMIT 20;
```

Tracks long-running operations such as schema changes, backups, and restores.

### Users

```sql
SELECT * FROM system.users;
```

Lists SQL users.

### Role Members

```sql
SELECT * FROM system.role_members;
```

Shows role memberships.

### Settings

```sql
SELECT name, value FROM system.settings LIMIT 20;
```

Displays cluster-level configuration settings.

### Comments

```sql
SELECT * FROM system.comments LIMIT 10;
```

Stores comments added to schema objects.

---

## Complete Flow

```
Application

        │

        ▼

Gateway Node

10.10.1.10

        │

        ▼

Leaseholder

10.10.2.10

        │

        ▼

Raft Consensus

        │

 ┌──────┴─────────┐

 ▼                ▼

10.10.1.10     10.10.3.10

Replica         Replica

        │

 Majority ACK (2 of 3)

        │

        ▼

 Commit

        │

        ▼

 Client receives SUCCESS
```

