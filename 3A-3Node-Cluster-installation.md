### Rebuiling 3 Node Cluster 

Since your current setup uses `start-single-node`, **remove the existing cluster first**, 
then build a proper distributed cluster.

We will first create a **2-node cluster**:

* **Node1:** `oel9-n1` → `10.10.10.11`
* **Node2:** `oel9-n2` → `10.10.10.12`

Later we will add:

* **Node3:** `oel9-n3` → `10.10.10.13`

> **Note:** A 2-node CockroachDB cluster is suitable for learning,
> but it's **not recommended for production** because Raft requires a majority. With 2 replicas,
> losing one node means the cluster cannot maintain quorum. Production deployments should use at least **3 nodes**.

---

### Step 1: Stop CockroachDB (Node1)

```bash
sudo systemctl stop cockroach
```

Verify:

```bash
sudo systemctl status cockroach
```

---

### Step 2: Disable Service (Optional)

```bash
sudo systemctl disable cockroach
```

---

### Step 3: Remove Existing Data

⚠️ This deletes the existing single-node cluster.

```bash
sudo rm -rf /var/lib/cockroach/data/*
```

Verify:

```bash
ls -lha /var/lib/cockroach/data
```

The directory should be empty except possibly for `lost+found` if it's a separate filesystem.

---

### Step 4: Remove Old Certificates

```bash
sudo rm -rf /var/lib/cockroach/certs/*
```

---

### Step 5: Remove Old CA Key

```bash
sudo rm -rf /var/lib/cockroach/my-safe-directory/*
```

---

### Step 6: Verify Cleanup

```bash
ls -l /var/lib/cockroach/data

ls -l /var/lib/cockroach/certs

ls -l /var/lib/cockroach/my-safe-directory
```

All three directories should be empty.

---

After this, we have to proceed with below steps

* **Step 7:** Generate a new CA.
* **Step 8:** Generate node certificates for **10.10.10.11** and **10.10.10.12**.
* **Step 9:** Copy the required certificates to Node2.
* **Step 10:** Create new `systemd` service files using `cockroach start` (not `start-single-node`).
* **Step 11:** Start both nodes.
* **Step 12:** Run `cockroach init`.
* **Step 13:** Verify the 2-node cluster.

Later we will add **Node3 (10.10.10.13)** and explain how CockroachDB rebalances ranges and replicas after a new node joins.

### Lab Environment

| Node  | Hostname  | Host-Only IP  | NAT IP            |
| ----- | --------- | ------------- | ----------------- |
| Node1 | `oel9-n1` | `10.10.10.11` | `192.168.235.247` |
| Node2 | `oel9-n2` | `10.10.10.12` | `192.168.235.248` |

We will use **Host-Only IPs** (`10.10.10.x`) for cluster communication.

> **Who runs what?**
>
> * **`venkat` user** → OS administration (`sudo`, copy files, edit systemd, start/stop services)
> * **`cockroach` user** → Generate certificates and run CockroachDB CLI commands

---

### Step 7 - Generate Certificates (Node1 Only)

## Login

**Node1**

```text
Hostname : oel9-n1
User     : venkat
```
Switch to the root user:

```bash
sudo -i
passwd cockroach
```
set password like cock123


then Switch to the CockroachDB user:

```bash
sudo su - cockroach
```

Create the CA key directory:

```bash
mkdir -p /var/lib/cockroach/my-safe-directory
```

Generate the CA:

```bash
cockroach cert create-ca \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

### Generate Node1 Certificate

Still on **Node1** as **cockroach**:

```bash
cockroach cert create-node \
  localhost \
  127.0.0.1 \
  oel9-n1 \
  10.10.10.11 \
  192.168.235.247 \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

### Generate Node2 Certificate

Still on **Node1** as **cockroach**.

Generate Node2's certificate **before copying it**:

First create the directory:

```bash
mkdir -p /var/lib/cockroach/certs/node2
```

```bash
cockroach cert create-node \
  localhost \
  127.0.0.1 \
  oel9-n2 \
  10.10.10.12 \
  192.168.235.248 \
  --certs-dir=/var/lib/cockroach/certs/node2 \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

This creates:

```text
node.crt
node.key
```

for **Node2** inside:

```text
/var/lib/cockroach/certs/node2/
```

---

### Generate Root Client Certificate

Still on **Node1**:

```bash
cockroach cert create-client root \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

## Step 8 - Copy Certificates to Node2

Login as:

```text
Node1
User : venkat
```

Copy **Node2** certificates.

On **Node2**, create the directories first, already we created. make sure once

```bash
sudo mkdir -p /var/lib/cockroach/certs
sudo chown -R cockroach:cockroach /var/lib/cockroach
```

Now from **Node1**:

```bash
scp /var/lib/cockroach/certs/ca.crt \
    venkat@10.10.10.12:/tmp/
```

```bash
scp /var/lib/cockroach/certs/node2/node.crt \
    venkat@10.10.10.12:/tmp/
```

```bash
scp /var/lib/cockroach/certs/node2/node.key \
    venkat@10.10.10.12:/tmp/
```

```bash
scp /var/lib/cockroach/certs/client.root.crt \
    venkat@10.10.10.12:/tmp/
```

```bash
scp /var/lib/cockroach/certs/client.root.key \
    venkat@10.10.10.12:/tmp/
```

Then login to **Node2**:

```text
Hostname : oel9-n2
User     : venkat
```

Move them:

```bash
sudo mv /tmp/*.crt /var/lib/cockroach/certs/
sudo mv /tmp/*.key /var/lib/cockroach/certs/
```

Fix permissions:

```bash
sudo chown cockroach:cockroach /var/lib/cockroach/certs/*
```

```bash
sudo chmod 644 /var/lib/cockroach/certs/*.crt
```

```bash
sudo chmod 600 /var/lib/cockroach/certs/*.key
```

---

### Step 9 - Create Systemd Service

Do this on **both Node1 and Node2** as **venkat**.

Create:

```bash
sudo vi /etc/systemd/system/cockroach.service
```

Node1:

```ini
[Unit]
Description=CockroachDB Cluster
After=network.target

[Service]
Type=notify
User=cockroach
Group=cockroach

ExecStart=/usr/local/bin/cockroach start \
  --certs-dir=/var/lib/cockroach/certs \
  --store=/var/lib/cockroach/data \
  --listen-addr=10.10.10.11:26257 \
  --advertise-addr=10.10.10.11:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=10.10.10.11:26257,10.10.10.12:26257

Restart=always

[Install]
WantedBy=multi-user.target
```

Node2:

```ini
[Unit]
Description=CockroachDB Cluster
After=network.target

[Service]
Type=notify
User=cockroach
Group=cockroach

ExecStart=/usr/local/bin/cockroach start \
  --certs-dir=/var/lib/cockroach/certs \
  --store=/var/lib/cockroach/data \
  --listen-addr=10.10.10.12:26257 \
  --advertise-addr=10.10.10.12:26257 \
  --http-addr=0.0.0.0:8080 \
  --join=10.10.10.11:26257,10.10.10.12:26257

Restart=always

[Install]
WantedBy=multi-user.target
```

---

### Step 10 - Start Services

Run on **both nodes** as **venkat**:

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable cockroach
```

```bash
sudo systemctl start cockroach
```

Verify:

```bash
sudo systemctl status cockroach
```

---

### Step 11 - Initialize the Cluster

**Important:** Run this **only once**.

Run **only on Node1** as **cockroach**:

```bash
cockroach init \
  --certs-dir=/var/lib/cockroach/certs \
  --host=10.10.10.11
```

Expected:

```text
Cluster successfully initialized
```

---

### Step 12 - Verify Nodes

Run on **Node1**:

```bash
cockroach node status \
  --certs-dir=/var/lib/cockroach/certs \
  --host=10.10.10.11
```

You should see:

```text
node_id
-------
1
2
```

Or in SQL:

```bash
cockroach sql \
  --certs-dir=/var/lib/cockroach/certs \
  --host=10.10.10.11
```

Then:

```sql
SELECT node_id,address,is_live FROM crdb_internal.gossip_nodes;
```

---

### Step 13 - Verify Replication

```sql
SHOW RANGES FROM DATABASE system;
```

```sql
SELECT range_id,
       replicas,
       lease_holder
FROM crdb_internal.ranges
LIMIT 20;
```

Since this is a **2-node cluster**, you will initially see replicas based on your configured replication factor. Later, when you add **Node3** and increase `num_replicas` to 3, you will be able to demonstrate proper Raft quorum, replica distribution, and leaseholder balancing.

---

### Note:- One correction to the certificate generation

In the steps above, I used a separate directory (`certs/node2`) to avoid overwriting `node.crt` and `node.key`. That's the recommended approach when generating certificates for multiple nodes from the same CA. Each node must receive **its own** `node.crt` and `node.key`, while sharing the same `ca.crt` and (optionally) the same `client.root.*` certificates.

