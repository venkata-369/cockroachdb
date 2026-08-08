Here is the corrected and adjusted full guide based on your actual setup (you are running commands **on the node itself**, and `/var/lib/cockroach/` requires `sudo`):

---

## Step 1: Generate Certificates (Run on crdb-node3 = 10.10.3.10)

```bash
# Create directories (already done, skip if exists)
mkdir -p /home/ubuntu/certs /home/ubuntu/my-safe-directory

# Create CA certificate
cockroach cert create-ca \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Create node cert for Mumbai Node 1 (you are already on this node)
cockroach cert create-node \
  10.10.3.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Create certs dir and copy directly (no scp needed, you are on this node)
sudo mkdir -p /var/lib/cockroach/certs
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown -R cockroach:cockroach /var/lib/cockroach/certs
sudo chmod 700 /var/lib/cockroach/certs

# Delete local node cert before creating Node 2 cert
rm /home/ubuntu/certs/node.crt /home/ubuntu/certs/node.key

# Create node cert for Mumbai Node 2 (crdb-node4 = 10.10.4.10)
ubuntu@crdb-node4:~$ pwd
/home/ubuntu
mkdir -p /home/ubuntu/certs /home/ubuntu/my-safe-directory
cockroach cert create-node \
  10.10.4.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Prepare certs dir on Node 2 first
ssh ubuntu@10.10.4.10 "sudo mkdir -p /var/lib/cockroach/certs && sudo chown ubuntu:ubuntu /var/lib/cockroach/certs"

# SCP certs to Node 2
scp /home/ubuntu/certs/ca.crt \
    /home/ubuntu/certs/node.crt \
    /home/ubuntu/certs/node.key \
    ubuntu@10.10.4.10:/var/lib/cockroach/certs/

# Fix ownership on Node 2
ssh ubuntu@10.10.4.10 "sudo chown -R cockroach:cockroach /var/lib/cockroach/certs && sudo chmod 700 /var/lib/cockroach/certs"

# Create root client certificate
cockroach cert create-client root \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

[[Generate Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-2-generate-certificates)]

---

## Step 2: Repeat Certificate Generation for Singapore

Run on **crdb-node8 (10.30.2.151)**:

```bash
mkdir -p /home/ubuntu/certs /home/ubuntu/my-safe-directory

# Create a SEPARATE CA for Singapore cluster
cockroach cert create-ca \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Node cert for Singapore Node 1
cockroach cert create-node \
  10.30.2.151 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Copy directly on Node 1
sudo mkdir -p /var/lib/cockroach/certs
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown -R cockroach:cockroach /var/lib/cockroach/certs
sudo chmod 700 /var/lib/cockroach/certs

# Delete and create cert for Singapore Node 2
rm /home/ubuntu/certs/node.crt /home/ubuntu/certs/node.key

cockroach cert create-node \
  10.30.2.198 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key

# Prepare and copy to Singapore Node 2
ssh ubuntu@10.30.2.198 "sudo mkdir -p /var/lib/cockroach/certs && sudo chown ubuntu:ubuntu /var/lib/cockroach/certs"

scp /home/ubuntu/certs/ca.crt \
    /home/ubuntu/certs/node.crt \
    /home/ubuntu/certs/node.key \
    ubuntu@10.30.2.198:/var/lib/cockroach/certs/

ssh ubuntu@10.30.2.198 "sudo chown -R cockroach:cockroach /var/lib/cockroach/certs && sudo chmod 700 /var/lib/cockroach/certs"
```

[[Generate Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-2-generate-certificates)]

---

## Step 3: Update `/etc/default/cockroach` on Each Node

### Mumbai Node 1 (10.10.3.10)
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

### Mumbai Node 2 (10.10.4.10)
```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.10.4.10
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.10.3.10:26257,10.10.4.10:26257
LOCALITY=region=mumbai,zone=mumbai-4hub
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

### Singapore Node 1 (10.30.2.151)
```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.30.2.151
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.30.2.151:26257,10.30.2.198:26257
LOCALITY=region=singapore,zone=singapore-8hub
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

### Singapore Node 2 (10.30.2.198)
```bash
sudo tee /etc/default/cockroach <<EOF
NODE_IP=10.30.2.198
DATA_DIR=/var/lib/cockroach/data
LOG_DIR=/var/lib/cockroach/logs
JOIN_NODES=10.30.2.151:26257,10.30.2.198:26257
LOCALITY=region=singapore,zone=singapore-9hub
CERTS_DIR=/var/lib/cockroach/certs
EOF
```

---

## Step 4: Update `/etc/systemd/system/cockroach.service` on ALL Nodes

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

## Step 5: Stop, Wipe Data & Restart on ALL Nodes

> ⚠️ This **deletes all existing data**. Run on every node.

```bash
# Stop service
sudo systemctl stop cockroach

# Wipe existing insecure data
sudo rm -rf /var/lib/cockroach/data/*

# Reload and restart
sudo systemctl daemon-reload
sudo systemctl start cockroach
```

---

## Step 6: Reinitialize Both Clusters with Virtualization Flags

### Mumbai (Primary) — run once only
```bash
cockroach init \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10 \
  --virtualized
```

### Singapore (Standby) — run once only
```bash
cockroach init \
  --certs-dir=/home/ubuntu/certs \
  --host=10.30.2.151 \
  --virtualized-empty
```

[[PCR Init](https://www.cockroachlabs.com/docs/stable/set-up-physical-cluster-replication#step-1-create-the-primary-cluster)]

---

## ⚠️ Key Reminders

| Item | Detail |
|---|---|
| Minimum nodes | PCR requires **3 nodes minimum** per cluster — you currently have 2 |
| Data wipe required | Switching insecure → secure requires full reinitialization |
| No scp to self | You are on `10.10.3.10` — use `cp` not `scp` for Node 1 |
| CA exchange | For PCR, each cluster's nodes also need the **other cluster's `ca.crt`** |
| Separate CAs | Mumbai and Singapore should have their own CA certificates |
