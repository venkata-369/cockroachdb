### Singapore Cluster Certificate Setup

### Step 1: SSH into Singapore Node 1 (10.30.2.151)

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<Singapore-Node1-Public-IP>
```

### Step 2: Create Certificate Directories

```bash
mkdir certs
mkdir my-safe-directory
```

### Step 3: Generate Singapore CA (New, Separate CA)

```bash
cockroach cert create-ca \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

### Step 4: Generate Node 1 Certificate

```bash
cockroach cert create-node \
  10.30.2.151 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```
Generate Root Client Certificate on crdb-node8
```
cockroach cert create-client root \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```
Expected
```
ubuntu@crdb-node8:~$ ls -lrt certs/
total 20
-rw-r--r-- 1 ubuntu ubuntu 1151 Aug  9 02:03 ca.crt
-rw------- 1 ubuntu ubuntu 1675 Aug  9 02:03 node.key
-rw-r--r-- 1 ubuntu ubuntu 1204 Aug  9 02:03 node.crt
-rw-r--r-- 1 ubuntu ubuntu 1147 Aug  9 02:44 client.root.crt
-rw------- 1 ubuntu ubuntu 1675 Aug  9 02:44 client.root.key
```
```
| Certificate | Location | Used By |
|---|---|---|
| `ca.crt` | `/var/lib/cockroach/certs` | CockroachDB **service** (cockroach user) |
| `node.crt` + `node.key` | `/var/lib/cockroach/certs` | CockroachDB **service** (cockroach user) |
| `client.root.crt` + `client.root.key` | `/home/ubuntu/certs` | **CLI commands** run by ubuntu user |
```
[[Required Keys and Certificates Documention](https://www.cockroachlabs.com/docs/stable/authentication#using-cockroach-cert-or-openssl-commands)]
```
---
### So the Flow is:

- `/var/lib/cockroach/certs` → used by the **CockroachDB service** to run the node
- `/home/ubuntu/certs` → used by **you** when running CLI commands like:
  - `cockroach init`
  - `cockroach sql`
  - `cockroach node status`

### Step 5: Copy Certs to `/var/lib/cockroach/certs/` on Node 1

```bash
sudo mkdir -p /var/lib/cockroach/certs
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown -R cockroach:cockroach /var/lib/cockroach/certs/
sudo chmod 700 /var/lib/cockroach/certs/
sudo chmod 600 /var/lib/cockroach/certs/node.key
sudo chmod 644 /var/lib/cockroach/certs/ca.crt
sudo chmod 644 /var/lib/cockroach/certs/node.crt
```

---

## Copy CA to Singapore Node 2 (10.30.2.198)
### On Singapore Node 2 
```
mkdir -p /home/ubuntu/certs
mkdir -p /home/ubuntu/my-safe-directory
```
### From Your Local Machine:

```bash
# Download CA from Singapore Node 1
scp -i ~/.ssh/id_rsa \
  ubuntu@<Singapore-Node1-Public-IP>:/home/ubuntu/certs/ca.crt \
  ~/sg-ca.crt

scp -i ~/.ssh/id_rsa \
  ubuntu@<Singapore-Node1-Public-IP>:/home/ubuntu/my-safe-directory/ca.key \
  ~/sg-ca.key

# Upload CA to Singapore Node 2
scp -i ~/.ssh/id_rsa \
  ~/sg-ca.crt \
  ubuntu@<Singapore-Node2-Public-IP>:/home/ubuntu/certs/ca.crt

scp -i ~/.ssh/id_rsa \
  ~/sg-ca.key \
  ubuntu@<Singapore-Node2-Public-IP>:/home/ubuntu/my-safe-directory/ca.key
```

### On Singapore Node 2 — Generate Node 2 Certificate

```bash
cockroach cert create-node \
  10.30.2.198 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

### Copy Certs to `/var/lib/cockroach/certs/` on Node 2

```bash
sudo mkdir -p /var/lib/cockroach/certs
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown -R cockroach:cockroach /var/lib/cockroach/certs/
sudo chmod 700 /var/lib/cockroach/certs/
sudo chmod 600 /var/lib/cockroach/certs/node.key
sudo chmod 644 /var/lib/cockroach/certs/ca.crt
sudo chmod 644 /var/lib/cockroach/certs/node.crt
```

---

#### ⚠️ Key Points
### verify on Node 1
```
cockroach cert list --certs-dir=/home/ubuntu/certs
```
Expected
```
ubuntu@crdb-node8:~$ cockroach cert list --certs-dir=/home/ubuntu/certs
Certificate directory: /home/ubuntu/certs
  Usage | Certificate File | Key File |  Expires   |                   Notes                    | Error
--------+------------------+----------+------------+--------------------------------------------+--------
  CA    | ca.crt           |          | 2036/08/16 | num certs: 1                               |
  Node  | node.crt         | node.key | 2031/08/13 | addresses: localhost,10.30.2.151,127.0.0.1 |
(2 rows)
```
```
sudo cockroach cert list --certs-dir=/var/lib/cockroach/certs 
```
Expected
```
ubuntu@crdb-node8:~$ sudo cockroach cert list --certs-dir=/var/lib/cockroach/certs
Certificate directory: /var/lib/cockroach/certs
  Usage | Certificate File | Key File |  Expires   |                   Notes                    | Error
--------+------------------+----------+------------+--------------------------------------------+--------
  CA    | ca.crt           |          | 2036/08/16 | num certs: 1                               |
  Node  | node.crt         | node.key | 2031/08/13 | addresses: localhost,10.30.2.151,127.0.0.1 |
(2 rows)
```

### verify on Node 2
```
cockroach cert list --certs-dir=/home/ubuntu/certs
```
Expected
```
ubuntu@crdb-node9:~$ cockroach cert list \
  --certs-dir=/home/ubuntu/certs
Certificate directory: /home/ubuntu/certs
  Usage | Certificate File | Key File |  Expires   |                   Notes                    | Error
--------+------------------+----------+------------+--------------------------------------------+--------
  CA    | ca.crt           |          | 2036/08/16 | num certs: 1                               |
  Node  | node.crt         | node.key | 2031/08/13 | addresses: localhost,10.30.2.198,127.0.0.1 |
(2 rows)
```
```
sudo cockroach cert list --certs-dir=/var/lib/cockroach/certs 
```
Expected
```
ubuntu@crdb-node9:~$ sudo cockroach cert list \
  --certs-dir=/var/lib/cockroach/certs
Certificate directory: /var/lib/cockroach/certs
  Usage | Certificate File | Key File |  Expires   |                   Notes                    | Error
--------+------------------+----------+------------+--------------------------------------------+--------
  CA    | ca.crt           |          | 2036/08/16 | num certs: 1                               |
  Node  | node.crt         | node.key | 2031/08/13 | addresses: localhost,10.30.2.198,127.0.0.1 |
(2 rows)
```
```
| Item | Detail |
|---|---|
| Singapore CA | Must be **separate** from Mumbai CA |
| Same SSH key (`crdb_key`) | ✅ Fine — SSH key is unrelated to CockroachDB certs |
| Node 1 IP | `10.30.2.151` |
| Node 2 IP | `10.30.2.198` |
| Delete temp files | Remove `~/sg-ca.crt` and `~/sg-ca.key` from local machine after copying |
```
---
## ✅ Node 2 (crdb-node9) `/var/lib/cockroach/certs` is Correct!

| Item | Value | Status |
|---|---|---|
| CA expires | 2036/08/16 | ✅ Valid |
| Node cert expires | 2031/08/13 | ✅ Valid |
| Node address | `10.30.2.198` | ✅ Correct IP |
| Error | empty | ✅ No errors |

---

### ✅ Singapore Cluster Certificate Status

| Node | IP | Certs in `/var/lib/cockroach/certs` |
|---|---|---|
| Node 1 (crdb-node8) | 10.30.2.151 | ✅ Done |
| Node 2 (crdb-node9) | 10.30.2.198 | ✅ Done |

---

### ⏭️ Next Steps: Update /etc/default/cockroach Files on Both Singapore Nodes

### On Singapore Node 1 (crdb-node8 — 10.30.2.151):

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.30.2.151
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.30.2.151:26257,10.30.2.198:26257
LOCALITY=region=ap-southeast-1,zone=ap-southeast-1a
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

### On Singapore Node 2 (crdb-node9 — 10.30.2.198):

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.30.2.198
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.30.2.151:26257,10.30.2.198:26257
LOCALITY=region=ap-southeast-1,zone=ap-southeast-1b
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

Then verify on each node:

```bash
cat /etc/default/cockroach
```
---



## ⏭️ Next: Update systemd Service File on Both Singapore Nodes

Run this on **both crdb-node8 and crdb-node9**:

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

[[Deploy On-Premises](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-3-start-nodes)]

---

## Verify Service File on Both Nodes

```bash
systemctl cat cockroach
```

Confirm:
- ✅ `--certs-dir=${CERTS_DIR}` is present
- ❌ `--insecure` is **NOT** present

---

### Then Start Both Singapore Nodes

Run on **both crdb-node8 and crdb-node9**:

```bash
sudo systemctl stop cockroach
sudo rm -rf /var/lib/cockroach/data/*
sudo systemctl daemon-reload
sudo systemctl start cockroach
sudo systemctl status cockroach
```

---

### Then Initialize Singapore Cluster (Once Only on crdb-node8)

```bash
cockroach init --certs-dir=/home/ubuntu/certs --host=10.30.2.151
```

You should see:
```
Cluster successfully initialized
```

### ✅ Singapore Cluster is Fully Operational!

Both nodes are live and healthy: [[cockroach node status CockroachDB Documention](https://www.cockroachlabs.com/docs/stable/cockroach-node#identify-live-nodes-in-an-unavailable-cluster)]

| Node | Address | Locality | Available | Live |
|---|---|---|---|---|
| 1 | 10.30.2.151:26257 | ap-southeast-1a | ✅ true | ✅ true |
| 2 | 10.30.2.198:26257 | ap-southeast-1b | ✅ true | ✅ true |

---

### ✅ Overall Cluster Status Summary

| Cluster | Region | Nodes | Status |
|---|---|---|---|
| Singapore (Primary) | ap-southeast-1 | crdb-node8, crdb-node9 | ✅ Ready |
| Mumbai (Standby) | ap-south-1 | crdb-node3, crdb-node4 | ✅ Ready |



