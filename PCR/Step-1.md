Note:- While working on AWS Cluster installation, we create a crdb_key, that you can download to your local ca certificate. that certificate you can copy in all the nodes. you find the below steps.

#### Summary:-

1, Any Node Create CA Certificate(Unique in all the nodes), already we created in AWS.
2, Copy in all the nodes
3, Create a Node Certificate on every Node
4, Create a Client Certificate on every node to connect through SQL

### Copy ca.crt files Directly from Local Machine

You need the **Communication Between Node 3 to Node 4 using public IP**. Check your AWS Console for it, then run these commands **from your local Git Bash**:

### ca.crt file should be unique in all the nodes to Ensure Secure 

Node 1

```
mkdir -p certs
mkdir -p my-safe-directory
cockroach cert create-ca --certs-dir=/home/ubuntu/certs --ca-key=/home/ubuntu/my-safe-directory/ca.key
```
WorkLog

```
ubuntu@crdb-node1:~$ mkdir -p certs
ubuntu@crdb-node1:~$ mkdir -p my-safe-directory
ubuntu@crdb-node1:~$ cockroach cert create-ca \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

### Step 1: Download ca.crt and ca.key from Node 1

```bash
scp -i ~/.ssh/id_rsa \
  ubuntu@13.207.56.135:/home/ubuntu/certs/ca.crt \
  ~/ca.crt

scp -i ~/.ssh/id_rsa \
  ubuntu@13.207.56.135:/home/ubuntu/my-safe-directory/ca.key \
  ~/ca.key
```

### Step 2: Upload ca.crt and ca.key to Node 2

```bash
# Replace <Node4-Public-IP> with actual public IP of crdb-node4
scp -i ~/.ssh/id_rsa \
  ~/ca.crt \
  ubuntu@<Node4-Public-IP>:/home/ubuntu/certs/

scp -i ~/.ssh/id_rsa \
  ~/ca.key \
  ubuntu@<Node2-Public-IP>:/home/ubuntu/my-safe-directory/
```

[[AWS Certificates](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-aws#step-5-generate-certificates)]

---

### Step 3: Verify Files on Node 2

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<Node2-Public-IP> "ls -lrt /home/ubuntu/certs/ && ls -lrt /home/ubuntu/my-safe-directory/"
```

You should see:
```
/home/ubuntu/certs/ca.crt
/home/ubuntu/my-safe-directory/ca.key
```
---
### Step 4: Generate Node Certs on Node 1
```
cockroach cert create-node \
  10.10.1.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

### Step 4: Generate Node Certs on Node 2

SSH into Node 2 and run:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<Node2-Public-IP>

cockroach cert create-node \
  10.10.4.10 \
  localhost \
  127.0.0.1 \
  --certs-dir=/home/ubuntu/certs \
  --ca-key=/home/ubuntu/my-safe-directory/ca.key
```

---

#### ⚠️ Key Points

| Item | Detail |
|---|---|
| Agent forwarding issue | Known Git Bash/Windows limitation — use direct SCP instead |
| Public IP of Node 4 | Check AWS Console → EC2 → Instances → `crdb-node4` |
| Same CA for all Mumbai nodes | Use the same `ca.crt` and `ca.key` generated on Node 3 |
| Delete temp files after | Remove `~/ca.crt` and `~/ca.key` from local machine after copying |
---

### Fix: Node 1 Certificate

```bash
# Verify ca.crt and ca.key are present
ls -lrt /home/ubuntu/certs/
ls -lrt /home/ubuntu/my-safe-directory/
```


[[Generate Certificates_CockroachDB Certificaates](https://www.cockroachlabs.com/docs/stable/cockroach-cert#examples)]

---

## Verify Files on all nodes the cluster

```bash
ls -lrt /home/ubuntu/certs/
```

You should see:
```
ca.crt
node.crt
node.key
```

---

### Then Copy to `/var/lib/cockroach/certs/`

```bash
sudo cp /home/ubuntu/certs/ca.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.crt /var/lib/cockroach/certs/
sudo cp /home/ubuntu/certs/node.key /var/lib/cockroach/certs/
sudo chown cockroach:cockroach /var/lib/cockroach/certs/ca.crt
sudo chown cockroach:cockroach /var/lib/cockroach/certs/node.crt
sudo chown cockroach:cockroach /var/lib/cockroach/certs/node.key
sudo chmod 700 /var/lib/cockroach/certs
```

---

### Verify Final State on Node 1

```bash
sudo ls -lrt /var/lib/cockroach/certs/
```

You should see:
```
ca.crt   (owned by cockroach)
node.crt (owned by cockroach)
node.key (owned by cockroach)
```

---

#### ⚠️ Key Point

| Issue | Detail |
|---|---|
| `node.crt` missing on Node 3 | Deleted earlier to create Node 2 cert — must regenerate with Node 3's IP |
| Each node has unique cert | Node 3 cert uses `10.10.1.10`, Node 2 cert uses `10.10.2.10` |
| `ca.crt` is shared | Same `ca.crt` used across all Mumbai nodes |

```
sudo cockroach cert list --certs-dir=/var/lib/cockroach/certs
```
Expected Output
```
ubuntu@crdb-node2:~$ sudo cockroach cert list --certs-dir=/var/lib/cockroach/certs
Certificate directory: /var/lib/cockroach/certs
  Usage | Certificate File | Key File |  Expires   |                   Notes                   | Error
--------+------------------+----------+------------+-------------------------------------------+--------
  CA    | ca.crt           |          | 2036/08/15 | num certs: 1                              |
  Node  | node.crt         | node.key | 2031/08/13 | addresses: localhost,10.10.2.10,127.0.0.1 |
(2 rows)
```
