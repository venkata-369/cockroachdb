



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

## Then Initialize Singapore Cluster (Once Only on crdb-node8)

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

---

### ⏭️ Next: Set Up PCR (Physical Cluster Replication)

Now that both clusters are running, the next steps are:

1. **Exchange CA certificates** between Singapore and Mumbai clusters
2. **Generate replication user** on Singapore (primary)
3. **Generate encoded connection string** using `cockroach encode-uri`
4. **Start replication** from Mumbai (standby) using `START VIRTUAL CLUSTER`

