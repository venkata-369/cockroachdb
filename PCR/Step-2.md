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

### Step 1: Generate Node Cert on Node 4

SSH into Node 4 and run:

```bash
cockroach cert create-node \
  10.10.4.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

Verify files:

```bash
ls -lrt /home/ubuntu/certs/
```

You should see:
```
ca.crt
node.crt
node.key
```

But failed 
```
ubuntu@crdb-node4:~$ cockroach cert create-node \
  10.10.4.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
ERROR: connection lost.

failed to generate node certificate and key: error writing node server certificate to /home/ubuntu/certs/node.crt: open /home/ubuntu/certs/node.crt: file exists
Failed running "cert create-node"
ubuntu@crdb-node4:~$ ls -lrt /home/ubuntu/certs/
total 12
-rw-r--r-- 1 ubuntu ubuntu 1151 Aug  8 23:38 ca.crt
-rw------- 1 ubuntu ubuntu 1679 Aug  8 23:40 node.key
-rw-r--r-- 1 ubuntu ubuntu 1204 Aug  8 23:40 node.crt
ubuntu@crdb-node4:~$ rm /home/ubuntu/certs/node.crt /home/ubuntu/certs/node.key
```

---
The error is clear — `node.crt` and `node.key` **already exist** in `/home/ubuntu/certs/` on Node 4. This happened because the files were copied over from Node 3 earlier.

You actually **don't need to run `cockroach cert create-node` again** — the files are already there. But these are **Node 3's certificates**, not Node 4's. You need to delete them and regenerate.

---

## Fix: Delete Existing Node Cert and Regenerate

```bash
# Delete the old node cert (these belong to Node 3)
rm /home/ubuntu/certs/node.crt /home/ubuntu/certs/node.key

# Now generate Node 4's certificate
cockroach cert create-node \
  10.10.4.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

### Verify New Certs

```bash
ls -lrt /home/ubuntu/certs/
```

You should see fresh timestamps on `node.crt` and `node.key`:

```
ca.crt
node.crt
node.key
```

---

[[CockroachDB Doc Generate Certificates](https://www.cockroachlabs.com/docs/stable/cockroach-cert#examples)]

---

### Step 2: Copy Certs to `/var/lib/cockroach/certs/` on Node 4

```bash
sudo mkdir -p /var/lib/cockroach/certs
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown cockroach:cockroach /var/lib/cockroach/certs/ca.crt
sudo chown cockroach:cockroach /var/lib/cockroach/certs/node.crt
sudo chown cockroach:cockroach /var/lib/cockroach/certs/node.key
sudo chmod 700 /var/lib/cockroach/certs
```

[[Generate Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-2-generate-certificates)]

---

### Step 3: Create Root Client Certificate (on Node 3)

Go back to Node 3 and run:

```bash
cockroach cert create-client root \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

dafdasfsdfasdfsadfdsfsadfadsfadsfdsfdsfsdfsdafadsfsdfadsfsdfasfdsfasdfadsfasdfsssssssssssssssssssssssss
Yes, the certificates look **correct**! Here is what to confirm:

---

### ✅ Certificate Verification

| Item | Node 3 | Node 4 | Status |
|---|---|---|---|
| CA cert | `ca.crt` expires 2036/08/15 | `ca.crt` expires 2036/08/15 | ✅ Same CA |
| Node cert | expires 2031/08/13 | expires 2031/08/12 | ✅ Valid |
| Node address | `10.10.3.10` | `10.10.4.10` | ✅ Correct IPs |
| Error column | empty | empty | ✅ No errors |

[[List Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-2-generate-certificates)]

---

### ✅ What This Confirms

- Both nodes have the **same CA** (`ca.crt`) — required for nodes to trust each other
- Each node has its **own unique node cert** with the correct IP address
- **No errors** in either certificate
- Certificates are **valid for many years**

---

### Step 4: Update /etc/default/cockroach Files on BOTH Mumbai Nodes

Run on **Each Mumbai node**:

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.3.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.3.10:26257,10.10.4.10:26257
LOCALITY=region=ap-south-1,zone=ap-south-1a
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.4.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.3.10:26257,10.10.4.10:26257
LOCALITY=region=ap-south-1,zone=ap-south-1b
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

> ⚠️ This deletes all existing insecure data. Run on **both Node 3 and Node 4**:

```bash
sudo systemctl stop cockroach
sudo rm -rf /var/lib/cockroach/data/*
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
ubuntu@crdb-node3:~$ cockroach node status \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10
  id |     address      |   sql_address    |  build  |              started_at              |              updated_at              |              locality              | is_available | is_live
-----+------------------+------------------+---------+--------------------------------------+--------------------------------------+------------------------------------+--------------+----------
   1 | 10.10.3.10:26257 | 10.10.3.10:26257 | v25.2.2 | 2026-08-09 01:32:50.498025 +0000 UTC | 2026-08-09 01:35:32.552168 +0000 UTC | region=ap-south-1,zone=ap-south-1a | true         | true
   2 | 10.10.4.10:26257 | 10.10.4.10:26257 | v25.2.2 | 2026-08-09 01:32:50.810229 +0000 UTC | 2026-08-09 01:35:32.83211 +0000 UTC  | region=ap-south-1,zone=ap-south-1b | true         | true
(2 rows)
```

---

### ✅ Mumbai Cluster is Fully Operational!

Both nodes are live and healthy:

| Node | Address | Locality | Available | Live |
|---|---|---|---|---|
| 1 | 10.10.3.10:26257 | ap-south-1a | ✅ true | ✅ true |
| 2 | 10.10.4.10:26257 | ap-south-1b | ✅ true | ✅ true |

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
| Singapore (Node 1 & 2) | ap-southeast-1 | **Primary** (active, serves traffic) |
| Mumbai (Node 3 & 4) | ap-south-1 | **Standby** (passive, receives replication) |

---


