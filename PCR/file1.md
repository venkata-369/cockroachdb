Great! Now that the CA files are on Node 4, here is what to do next:

---

## Step 1: Generate Node Cert on Node 4

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

[[Generate Certificates](https://www.cockroachlabs.com/docs/stable/cockroach-cert#examples)]

---

## Verify New Certs

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

## Then Copy to `/var/lib/cockroach/certs/`

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

---

## ⚠️ Key Point

| Issue | Detail |
|---|---|
| `file exists` error | Old `node.crt`/`node.key` from Node 3 were copied over — delete them first |
| Each node needs its own cert | `node.crt` must be generated **per node** with that node's IP |
| `ca.crt` is shared | Keep `ca.crt` — it is the same for all Mumbai nodes |

[[Generate Certificates](https://www.cockroachlabs.com/docs/stable/cockroach-cert#examples)]

---

## Step 2: Copy Certs to `/var/lib/cockroach/certs/` on Node 4

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

## Step 3: Create Root Client Certificate (on Node 3)

Go back to Node 3 and run:

```bash
cockroach cert create-client root \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

---

## Step 4: Update Service Files on BOTH Mumbai Nodes

Run on **each Mumbai node**:

```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.3.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.3.10:26257,10.10.4.10:26257
LOCALITY=region=mumbai,zone=mumbai-3hub
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

> ⚠️ Change `NODE_IP` and `LOCALITY` accordingly for Node 4.

Update the systemd service on **each node**:

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
  --log-dir=\${LOG_DIR}

Restart=always
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
EOF
```

---

## Step 5: Stop, Wipe Data & Restart on Both Mumbai Nodes

> ⚠️ This deletes all existing insecure data.

```bash
sudo systemctl stop cockroach
sudo rm -rf /var/lib/cockroach/data/*
sudo systemctl daemon-reload
sudo systemctl start cockroach
```

---

## Step 6: Initialize Mumbai Cluster

Run **once only** from Node 3:

```bash
cockroach init \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10 \
  --virtualized
```

---

## What's Next After Mumbai

| Task | Detail |
|---|---|
| ✅ Mumbai Node 3 | Certs done |
| ✅ Mumbai Node 4 | Certs done (just completed) |
| ⏳ Singapore Node 1 & 2 | Repeat cert generation with a **separate CA** for Singapore cluster |
| ⏳ PCR Setup | Exchange CA certs between clusters after both are initialized |
