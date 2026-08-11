### CockroachDB installation on OEL 9
> Replace `24.3.x` with the exact v24 release you want (for example, `24.3.21`).

### Step 1: Update the server

```bash
sudo dnf update -y
```

### Step 2: Install required packages

```bash
sudo dnf install -y wget curl tar unzip
```

### Step 3: Download CockroachDB v24

```bash
wget https://binaries.cockroachdb.com/cockroach-v24.3.21.linux-amd64.tgz
```

### Step 4: Extract the package

```bash
tar -xzf cockroach-v24.3.21.linux-amd64.tgz
```

### Step 5: Copy the binary

```bash
sudo cp cockroach-v24.3.21.linux-amd64/cockroach /usr/local/bin/
```

### Step 6: Make it executable

```bash
sudo chmod +x /usr/local/bin/cockroach
```

### Step 7: Verify installation

```bash
cockroach version
```

### Step 8: Create CockroachDB user

```bash
sudo useradd -m -s /bin/bash cockroach
```

### Step 9: Create data directory

```bash
sudo mkdir -p /data/cockroach
```

### Step 10: Set ownership

```bash
sudo chown -R cockroach:cockroach /data/cockroach
```

### Step 11: Create certificates directory (optional for later)

```bash
sudo mkdir -p /data/cockroach/certs
```

### Step 12: Change ownership

```bash
sudo chown -R cockroach:cockroach /data/cockroach/certs
```

### Step 13: Open firewall ports

```bash
sudo firewall-cmd --permanent --add-port=26257/tcp
```

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
```

### Step 14: Reload firewall

```bash
sudo firewall-cmd --reload
```

### Step 15: Start a single-node cluster

```bash
cockroach start-single-node \
--insecure \
--store=/data/cockroach \
--listen-addr=0.0.0.0:26257 \
--http-addr=0.0.0.0:8080 \
--background
```

### Step 16: Initialize SQL (if required)

```bash
cockroach sql --insecure
```

### Step 17: Show databases

```sql
SHOW DATABASES;
```

### Step 18: Exit SQL

```sql
\q
```

### Step 19: Check node status

```bash
cockroach node status --insecure
```

### Step 20: Open the Web UI

```
http://<public_ip>:8080
```

---

### Check version

```bash
cockroach version
```

### Stop CockroachDB

```bash
cockroach quit --insecure
```

### Check running process

```bash
ps -ef | grep cockroach
```
