## 1. Introduction to CockroachDB 

### What is CockroachDB?

CockroachDB is a **cloud-native distributed SQL database** designed to provide:

* Horizontal scalability
* High availability
* Strong consistency (ACID)
* Automatic replication
* Automatic failover
* Geo-distributed deployments

Unlike traditional databases, CockroachDB distributes data automatically across multiple nodes while presenting a single SQL database to applications.

**Official Definition**

> CockroachDB is a distributed SQL database that combines the familiarity of SQL with the scalability and resilience of NoSQL systems.

---

### Why is it called CockroachDB?

Cockroaches are known for surviving disasters.

Similarly, CockroachDB is designed to survive:

* Server failures
* Disk failures
* Network failures
* Availability Zone failures
* Entire Region failures

without application downtime.

---

### Why do we need Distributed SQL?

Imagine an e-commerce company.

Initially:

```
Application

      |

   MySQL Server
```

This works well for a few thousand users.

As traffic grows:

* Millions of customers
* Thousands of orders per second
* Users across multiple countries

Problems appear:

* CPU reaches 100%
* Memory becomes full
* Storage runs out
* Replication lag
* Single point of failure

Scaling vertically (adding CPU/RAM) has limits.

Instead, distribute the workload across multiple servers.

```
Node1
Node2
Node3
Node4
```

Now the database can continue operating even if one node fails.

---

### Traditional Database Problems

Example: PostgreSQL

```
Application

      |

 PostgreSQL Primary

      |

 Standby
```

If the Primary server crashes:

* Failover is required.
* Applications reconnect.
* Downtime may occur.
* Manual intervention may be needed.

---

### CockroachDB Solution

```
Application

        |

 Load Balancer

   |     |     |

 Node1 Node2 Node3
```

Every node can accept SQL connections.

Advantages:

* No single primary server
* Automatic replication
* Automatic failover
* Automatic balancing

---

### Real-world Example

Suppose Amazon has users in:

* India
* USA
* Europe

Traditional Database:

```
India -----> USA Database
```

Every query from India travels to the USA.

Latency:

250–300 ms

CockroachDB:

```
India Region

 Node1

 Node2

 Node3

Replication

USA Region

 Node4

 Node5

 Node6
```

Indian users access nearby replicas, reducing latency while maintaining consistency.

---

## Key Features

### Distributed SQL

Applications continue using standard SQL:

```sql
SELECT * FROM customers;
```

No special API is required.

---

### Horizontal Scaling

Instead of upgrading one server:

```
8 CPU

↓

16 CPU

↓

32 CPU
```

Simply add more nodes:

```
Node1

Node2

Node3

Node4

Node5
```

CockroachDB automatically redistributes data.

---

### Automatic Replication

If the replication factor is 3:

```
Range 101

Replica 1 → Node1

Replica 2 → Node2

Replica 3 → Node3
```

Three copies are always maintained.

---

### Automatic Failover

If Node2 fails:

```
Node1 ✓

Node2 ✗

Node3 ✓
```

Raft elects a new leader (leaseholder) automatically.

Applications continue working.

---

### Strong Consistency

When data is committed:

```sql
INSERT INTO orders VALUES (1001,'Laptop');
```

The transaction is committed only after the required replicas acknowledge it.

This prevents stale reads and inconsistent data.

---

### SQL Compatibility

CockroachDB supports familiar SQL operations:

```sql
CREATE DATABASE company;

CREATE TABLE employees
(
    emp_id INT PRIMARY KEY,
    name STRING,
    salary INT
);

INSERT INTO employees
VALUES
(1,'John',50000);

SELECT * FROM employees;
```

Anyone familiar with PostgreSQL can start working with CockroachDB quickly.

---

### Use Cases - Industries Using CockroachDB

* Banking & FinTech
  - **Core Banking Systems:** Store customer accounts, balances, and transactions with strong ACID consistency across multiple regions.
  - **Real-Time Payments:** Handle high-volume payment processing, fund transfers, and transaction validation with low latency.
  - **Fraud Detection:** Maintain real-time transaction data for detecting suspicious activities and enforcing security rules.
  - **Multi-Region Banking:** Provide high availability and disaster recovery by replicating data across multiple data centers.
  - ***Digital Banking Applications:** Support online banking, mobile apps, and APIs requiring scalability and always-on availability.

* E-commerce
* Healthcare
* SaaS
* Telecom
* Gaming
* Logistics

These industries require databases that stay available 24×7.

---

### Why Companies Choose CockroachDB

* 99.99%+ availability
* Automatic failover
* Zero single point of failure
* Multi-region support
* Distributed transactions
* Elastic scaling
* Cloud-native architecture
* PostgreSQL-compatible SQL

---


