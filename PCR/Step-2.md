Note:- Before Starting the Replication (PCR), need License key for all 4 nodes.

### Adding License to all Node in all the Regions[Mumbai & Singapore]
For License register with [cockroach](https://www.cockroachlabs.com/) and get keys
```
SET CLUSTER SETTING enterprise.license = 'crl-0-EODQ6NQGGAQyEH0vajlANUndqwNHF/AFHQ86EJOHfsC09EVVuYk4nCukzvE';
SHOW CLUSTER SETTING enterprise.license;
```
### Adding organization Node all the Clusters
```
SHOW CLUSTER SETTING cluster.organization;
SET CLUSTER SETTING cluster.organization = 'djs-colo';
```

[[List Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-2-generate-certificates)]

---

### ✅ What This Confirms

- All the 3 nodes have the **Same CA** (`ca.crt`) — required for nodes to trust each other
- Each node has its **own unique node cert** with the correct IP address
- **No errors** in either certificate
- Certificates are **valid for many years**

---

### Step 4: Update /etc/default/cockroach Files on THREE (3) Mumbai Nodes

Run on **Each Mumbai node**:

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.1.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.1.10:26257,10.10.2.10:26257,10.10.3.10:26257
LOCALITY=region=ap-south-1,zone=ap-south-1a
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.2.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.2.10:26257,10.10.3.10:26257,10.10.1.10:26257
LOCALITY=region=ap-south-1,zone=ap-south-1b
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.3.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.3.10:26257,10.10.1.10:26257,10.10.2.10:26257
LOCALITY=region=ap-south-1,zone=ap-south-1c
CERTS_DIR=/var/lib/cockroach/certs
EOF
```


> ⚠️ Change `NODE_IP` and `LOCALITY` accordingly for Node 2.

Update the systemd service on **each node**:
### Node 1
```
sudo tee /etc/systemd/system/cockroach.service <<EOF
[Unit]
Description=CockroachDB Database
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=cockroach
Group=cockroach
EnvironmentFile=/etc/default/cockroach

ExecStart=/usr/local/bin/cockroach start \
  --certs-dir=\${CERTS_DIR} \
  --store=\${DATA_DIR} \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=\${NODE_IP}:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=\${JOIN_NODES} \
  --locality=\${LOCALITY} \
  --log-dir=/var/lib/cockroach/logs

Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```

### Node 2
```bash
sudo tee /etc/systemd/system/cockroach.service <<EOF
[Unit]
Description=CockroachDB Database
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=cockroach
Group=cockroach
EnvironmentFile=/etc/default/cockroach

ExecStart=/usr/local/bin/cockroach start \
  --certs-dir=\${CERTS_DIR} \
  --store=\${DATA_DIR} \
  --listen-addr=0.0.0.0:26257 \
  --advertise-addr=\${NODE_IP}:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=\${JOIN_NODES} \
  --locality=\${LOCALITY} \
  --log-dir=/var/lib/cockroach/logs

Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```
### For Node 3, Repeat about Step cockroach.service
---
**Checklist**
Verify the Full File
```
systemctl cat cockroach
```

Make sure you see **all these flags** in `ExecStart`:

| Flag | Expected Value |
|---|---|
| `--certs-dir` | `${CERTS_DIR}` |
| `--store` | `${DATA_DIR}` |
| `--listen-addr` | `0.0.0.0:26257` |
| `--advertise-addr` | `${NODE_IP}:26257` |
| `--http-addr` | `0.0.0.0:8080` |
| `--join` | `${JOIN_NODES}` |
| `--locality` | `${LOCALITY}` |
| `--insecure` | ❌ Must **NOT** be present |

---

### Step 5: Stop, Wipe Data & Restart on Both Mumbai Nodes

> ⚠️ This deletes all existing insecure data. if already we worked with insecure on AWS  Run on **both Node 1, Node 2 and Node 3**:

```bash
sudo systemctl stop cockroach
sudo rm -rf /var/lib/cockroach/data/*
sudo rm -rf /var/lib/cockroach/log/*
```
```
sudo mkdir -p /var/lib/cockroach/data
sudo mkdir -p /var/lib/cockroach/logs
sudo chown -R cockroach:cockroach /var/lib/cockroach/
```
sudo systemctl daemon-reload
sudo systemctl start cockroach
```
You should see `Active: active (running)` on both nodes.

---

## Step 6: Initialize Mumbai Cluster

Run **Once only** from Node 1:

```bash
sudo cockroach init --certs-dir=/var/lib/cockroach/certs --host=10.10.1.10
```
or
```
cockroach init --certs-dir=/home/ubuntu/certs --host=10.10.1.10
```

Expected Output
```
ubuntu@crdb-node3:~$ cockroach init --certs-dir=/home/ubuntu/certs --host=10.10.1.10
Cluster successfully initialized
```
List
```
cockroach node status --certs-dir=/home/ubuntu/certs --host=10.10.1.10
```
Expected Output
```
ubuntu@crdb-node1:~$ cockroach node status --certs-dir=/home/ubuntu/certs --host=10.10.1.10
  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              |              locality              | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+------------------------------------+--------------+----------
   1 | 10.10.1.10:26257 | 10.10.1.10:26257 | v25.2.2 | 2026-08-20 01:55:31.680657 +0000 UTC | 2026-08-20 03:59:34.718401 +0000 UTC | region=ap-south-1,zone=ap-south-1a | true         | true
   2 | 10.10.3.10:26257 | 10.10.3.10:26257 | v25.2.2 | 2026-08-20 01:56:00.706849 +0000 UTC | 2026-08-20 03:59:33.724515 +0000 UTC | region=ap-south-1,zone=ap-south-1c | true         | true
   3 | 10.10.2.10:26257 | 10.10.2.10:26257 | v25.2.2 | 2026-08-20 01:56:08.01581 +0000 UTC  | 2026-08-20 03:59:35.032321 +0000 UTC | region=ap-south-1,zone=ap-south-1b | true         | true
(3 rows)
```

---

### ✅ Mumbai Cluster is Fully Operational!

Both nodes are live and healthy:

| Node | Address | Locality | Available | Live |
|---|---|---|---|---|
| 1 | 10.10.1.10:26257 | ap-south-1a | ✅ true | ✅ true |
| 2 | 10.10.2.10:26257 | ap-south-1b | ✅ true | ✅ true |
| 3 | 10.10.3.10:26257 | ap-south-1c | ✅ true | ✅ true |

---

### ⏭️ Then What's Next — Singapore Cluster Setup

Now you need to set up the **Singapore cluster** (primary cluster for PCR). Here is the overall plan: [[PCR Setup](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication)]

### Singapore Cluster Steps:
1. **Generate certificates** on Singapore Node 1 (separate CA from Mumbai)
2. **Copy CA** to Singapore Node 2
3. **Generate node certs** on each Singapore node
4. **Copy certs** to `/var/lib/cockroach/certs/` on each node
5. **Update service files** on both Singapore nodes
6. **Start CockroachDB** on both nodes
7. **Initialize** Singapore cluster with `cockroach init`

---

#### PCR Architecture Quick Look

| Cluster | Region | Role |
|---|---|---|
| Mumbai (Node 1,2,3) | ap-southeast-1 | **Primary** (active, serves traffic) |
| Singapore (Node 5,6,7) | ap-south-1 | **Standby** (passive, receives replication) |

---


