**Step-by-step Installation of CockroachDB v24.x on Oracle Linux 9 (OEL 9)**

### 1. Verify OEL Version

```bash
cat /etc/os-release
uname -r
```

***

### 2. Update OS

```bash
sudo dnf update -y
sudo dnf install -y wget curl tar firewalld
```

***

### 3. Download CockroachDB 24.x

Replace the version if required.

```bash
cd /tmp

wget https://binaries.cockroachdb.com/cockroach-v24.3.5.linux-amd64.tgz

tar -xvzf cockroach-v24.3.5.linux-amd64.tgz
```

Copy binaries:

```bash
sudo cp cockroach-v24.3.5.linux-amd64/cockroach /usr/local/bin/

sudo chmod +x /usr/local/bin/cockroach
```

Verify:

```bash
cockroach version
```

Example output:

```text
Build Tag:        v24.3.5
```

---

### 4. Create CockroachDB User

```bash
sudo useradd -r -m -d /var/lib/cockroach -s /bin/bash cockroach
````

Create directories:

```bash
sudo mkdir -p /var/lib/cockroach/data
sudo mkdir -p /var/lib/cockroach/certs
sudo mkdir -p /var/lib/cockroach/logs

sudo chown -R cockroach:cockroach /var/lib/cockroach
```

***

### 5. Generate TLS Certificates

Switch user:

```bash
sudo su - cockroach
```

Create CA certificate:

```bash
mkdir -p /var/lib/cockroach/my-safe-directory

cockroach cert create-ca --certs-dir=/var/lib/cockroach/certs --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

#### Create Node Certificate

Replace `crdb01.company.com` with your hostname/IP.

Check hostname:

```bash
hostname -f
hostname -I
````

Generate node certificate:

```bash
cockroach cert create-node \
  localhost \
  127.0.0.1 \
  $(hostname -f) \
  $(hostname -I | awk '{print $1}') \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

#### Create Root Client Certificate

```bash
cockroach cert create-client root \
   --certs-dir=/var/lib/cockroach/certs \
   --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

***

### 6. Verify Certificates

```bash
ls -ltr /var/lib/cockroach/certs
```

Expected:

```text
ca.crt
node.crt
node.key
client.root.crt
client.root.key
```

***

### 7. Open Firewall

```bash
sudo firewall-cmd --permanent --add-port=26257/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Port usage:

* 26257 = SQL
* 8080 = Admin UI [\[mylinux.work\]](https://mylinux.work/guides/cockroachdb-setup/)

***

### 8. Create Systemd Service

Create:

```bash
sudo vi /etc/systemd/system/cockroach.service
```

Content:

```ini
[Unit]
Description=CockroachDB
After=network.target

[Service]
Type=notify
User=cockroach
Group=cockroach

ExecStart=/usr/local/bin/cockroach start-single-node \
  --certs-dir=/var/lib/cockroach/certs \
  --store=/var/lib/cockroach/data \
  --listen-addr=$(hostname -I | awk '{print $1}'):26257 \
  --http-addr=0.0.0.0:8080 \
  --cache=25% \
  --max-sql-memory=25%

Restart=always
RestartSec=10

LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

***

### 9. Start Service

```bash
sudo systemctl daemon-reload

sudo systemctl enable cockroach

sudo systemctl start cockroach
```

Verify:

```bash
sudo systemctl status cockroach
```

***

### 10. Check Listening Ports

```bash
ss -tulpn | grep 26257
ss -tulpn | grep 8080
```

***

### 11. Connect Securely

Login using TLS certs:

```bash
cockroach sql \
  --certs-dir=/var/lib/cockroach/certs \
  --host=$(hostname -I | awk '{print $1}')
```

Check cluster:

```sql
SHOW DATABASES;
```

***

### 12. Create Admin User (Recommended)

```sql
CREATE USER admin WITH PASSWORD 'Training@369';
GRANT admin TO root;
```

***

### 13. Access Web Console

Open:

```text
https://<SERVER-IP>:8080
```

The Admin UI uses the generated certificates for secure communication. [\[cockroachlabs.com\]](https://www.cockroachlabs.com/docs/stable/secure-a-cluster)

***

### Quick Validation Commands

```bash
cockroach node status \
--certs-dir=/var/lib/cockroach/certs \
--host=<SERVER-IP>:26257
```

```bash
cockroach sql \
--certs-dir=/var/lib/cockroach/certs \
--host=<SERVER-IP> \
-e "select version();"
```
