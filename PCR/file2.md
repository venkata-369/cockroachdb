## Singapore Cluster Certificate Setup

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

## ✅ Singapore Cluster Certificate Status

| Node | IP | Certs in `/var/lib/cockroach/certs` |
|---|---|---|
| Node 1 (crdb-node8) | 10.30.2.151 | ✅ Done |
| Node 2 (crdb-node9) | 10.30.2.198 | ✅ Done |

---

### ⏭️ Next Steps: Update Service Files on Both Singapore Nodes

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

