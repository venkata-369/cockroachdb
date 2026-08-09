### ✅ Both Singapore Service Files !

| Node | IP | Locality | Status |
|---|---|---|---|
| crdb-node8 | 10.30.2.151 | ap-southeast-1a | ✅ Done |
| crdb-node9 | 10.30.2.198 | ap-southeast-1b | ✅ Done |

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

## Then Initialize Singapore Cluster (Once Only on crdb-node8)

```bash
cockroach init --certs-dir=/home/ubuntu/certs --host=10.30.2.151
```

You should see:
```
Cluster successfully initialized
```
