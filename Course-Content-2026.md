## CockroachDB administrator Course 

### Module 1: Introduction to CockroachDB

* What is CockroachDB?
* Evolution of Databases
* SQL vs NoSQL vs NewSQL (Distributed SQL)
* Why CockroachDB?
* CockroachDB Features
* Editions (Core, Enterprise, Cloud)
* Use Cases
* Architecture Overview

---
### Module 2: Installation & Environment Setup

* Prerequisites
* Oracle Linux 9 Installation
* Install CockroachDB
* Single-Node Cluster (Insecure)
* Secure vs Insecure Mode
* Single-Node Cluster (Secure)
    * TLS Certificate Architecture
    * Create CA Certificate
    * Create Node Certificate
    * Create Client Certificate
    * Start CockroachDB Service
    * Configure Systemd
    * Configure Firewall
* DB Console
* SQL Client
---

### Module 3: Cockroachdb Distributed Architecture

* Distributed SQL Architecture
* Node Architecture
* Cluster Architecture
* Stores
* Key-Value Layer
* SQL Layer
* DistSQL
* Ranges
* Range Splits
* Range Merges
* Replicas
* Leaseholders
* Gossip Protocol
* Raft Consensus
* Metadata Tables
  
**Hands-on**
  - Explore Internal Metadata
  - Cluster Topology
* Three-Node Cluster Installation
* Cluster Verification
---

### Module 4: Distributed Storage Internals

### Storage Engine

* Pebble Storage Engine
* SSTables
* LSM Trees
* MemTables
* WAL
* Write Path
* Read Path

### MVCC

* MVCC Architecture
* Version Storage
* Garbage Collection
* Closed Timestamps

### Range Management

* Range Splits
* Range Merges
* Lease Transfers
* Automatic Rebalancing
* Automatic Sharding

### Replication Internals

* Replica Factor
* Quorum
* Leader Election
* Leaseholder Election
* Failover
* Replica Recovery

---
### Module 5: Security & User Management

* Users
* Password Policies
* Roles
* Role Hierarchy
* Grants
* Revokes
* RBAC
* Encryption
   - Encryption in Transit
   - Encryption at Rest
* Audit Logs
* Security Best Practices
---

### Module 6: Backup & Restore

* FULL BACKUPS
* Incremental Backup
* RESTORE
* Scheduled Backups
* Cloud Storage Backups
* Backup Encryption
* Backup Validation
---

### Module 7: Cluster Administration

* Production CheckList
     - https://www.cockroachlabs.com/docs/v26.2/recommended-production-settings
* Cluster Initialization & Settings
     - Overview
          - Horizantal Scaling
          - Vertical Scaling
* Cluster Configuration
* Node Management
     - Adding Node
     - Remoing Node 
* Node Decommission
* Node Recommission
* Cluster Upgrade
* Cluster Health
* Cluster Diagnostics (Manage Long-Running Queries)
* Licensing

---
### Module 8: Performance Tuning

* Transactions
     - MVCC Performance
* ACID Properties
* UPSERT
* IMPORT
* EXPORT
* Indexes
     - Secondary Indexes
     - Partial Indexes
     - Hash sharded Indexes
     - Vector Indexes 
* EXPLAIN
* EXPLAIN ANALYZE
* Query Optimizer
* Statistics
* Cost-Based Optimization
* Query Plans
* Session Settings
* Vectorized Execution
* Statement Diagnostics
* SQL Activity
* Hotspots
* Contention Analysis

---
### Module 9: Monitoring & Observability

* DB Console
* Metrics Dashboards
* Sessions Page
* Health Checks
* Statement & Transactions Page
* Network Page
* Transactions Page
* Jobs Page
* Schedules Page
* Advanced Debug Page
   - License
---

### Module 11: Multi-Region Cluster [AWS (or) GCP (or) Azure]

* Localities
* Regions
* Zones
* Survival Goals
* REGIONAL BY ROW
* REGIONAL BY TABLE
* GLOBAL Tables
* Geo-Partitioning
* Multi-Region SQL
* Latency Optimization
---
### Module 12: Cross-Cluster Replication (HA)

* Physical Replication
   - Overview
   - Configuration & Setup
   - Failover from Primary to Standby
   - Monitoring
* Logical Data Replication
   - Overview
   - Configuration & Setup
   - Monitor Logical Replication
---
### Module 13: Troubleshooting

* Node Down
* Cluster Down
* Disk Full
* High CPU
* High Memory
* High Latency
* Slow Queries
* Hot Ranges
* Leaseholder Imbalance
* Replica Problems
* Clock Skew
* Certificate Expiry
* Network Partition
* Backup Failures
* SQL Connection Problems
---

### Module 14: Automation & Kubernetes

* Containers
   - Docker / Podman Desktop

### Kubernetes

* StatefulSets
* Persistent Volumes
* Storage Classes
* Scaling
* Rolling Updates

### Infrastructure as Code

* Terraform
* Ansible (Overview)

#### CI/CD

* GitHub Actions
* Kubernetes Deployment Automation
---

### Module 15: Migration to CockroachDB
* PostgreSQL Compatibility
* Schema Migration
* Data Migration
* Validation
* Cutover Strategy
* Rollback Strategy
* Migration Best Practices
---

### Module 16: Production Best Practices
* Production Architecture
* Hardware Sizing
* CPU Recommendations
* Memory Recommendations
* Storage Recommendations
* Filesystem Recommendations
* Backup Strategy
* Monitoring Strategy
* Security Checklist
* Upgrade Strategy
* Maintenance Windows
* Disaster Recovery Planning
---

### Module 17: Enterprise Features
* Enterprise Licensing
* Scheduled Backups
* Changefeeds
* Row-Level TTL
* Multi-Region Enterprise Features
* Backup to Cloud Storage
* Enterprise Security Features
---

### Module 18: DBA Interview Questions & Real-Time Scenarios
   - Production Case Studies
      * Banking
      * E-Commerce
      * SaaS
      * Financial Services
      * Multi-Region Deployments
---

