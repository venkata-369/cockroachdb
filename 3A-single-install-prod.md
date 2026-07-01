### Step-by-step Installation of CockroachDB v24.x on Oracle Linux 9 (OEL 9)

---
## Prerequisites

| User          | Purpose                                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **venkat**    | Normal Linux user with `sudo` privileges. Used for OS configuration, package installation, firewall configuration, and service management. |
| **cockroach** | Dedicated CockroachDB service account. Used for certificate generation and running the CockroachDB service.                                |
| **root**      | Used only through `sudo` when administrative privileges are required.                                                                      |

---

## Step 1. Verify Oracle Linux Version

**Run As:** `venkat`

Verify the operating system version and kernel.

```bash
cat /etc/os-release
uname -r
```

---

## Step 2. Update the Operating System

**Run As:** `venkat`

Update all installed packages.

```bash
sudo dnf update -y
```

Verify whether the required packages are installed.

```bash
rpm -qa | grep -E "wget|curl|tar|firewalld"
```

If any package is missing, install it.

```bash
sudo dnf install -y wget curl tar firewalld
```

Enable and start the firewall service.

```bash
sudo systemctl enable --now firewalld
```

Verify the firewall status.

```bash
sudo systemctl status firewalld
```

---

## Step 3. Download and Install CockroachDB v24.x

**Run As:** `venkat`

Download CockroachDB.

```bash
cd /tmp

wget https://binaries.cockroachdb.com/cockroach-v24.3.5.linux-amd64.tgz

tar -xvzf cockroach-v24.3.5.linux-amd64.tgz
```

Install the binary.

```bash
sudo cp cockroach-v24.3.5.linux-amd64/cockroach /usr/local/bin/

sudo chmod +x /usr/local/bin/cockroach
```

Verify the installation.

```bash
which cockroach

cockroach version
```

---

## Step 4. Create the CockroachDB Service Account

**Run As:** `venkat`

Create the CockroachDB user.

```bash
sudo useradd -r -m -d /var/lib/cockroach -s /bin/bash cockroach
```

Create the required directories.

```bash
sudo mkdir -p /var/lib/cockroach/data
sudo mkdir -p /var/lib/cockroach/certs
sudo mkdir -p /var/lib/cockroach/logs
sudo mkdir -p /var/lib/cockroach/my-safe-directory
```

Assign ownership.

```bash
sudo chown -R cockroach:cockroach /var/lib/cockroach
```

Assign permissions.

```bash
sudo chmod 700 /var/lib/cockroach
sudo chmod 700 /var/lib/cockroach/my-safe-directory
```

Verify.

```bash
sudo ls -ld /var/lib/cockroach
sudo ls -ld /var/lib/cockroach/data
sudo ls -ld /var/lib/cockroach/certs
```

---

## Step 5. Generate the Certificate Authority (CA)

**Run As:** `cockroach`

Switch to the CockroachDB user.

```bash
sudo su - cockroach
```

Generate the CA certificate.

```bash
cockroach cert create-ca \
    --certs-dir=/var/lib/cockroach/certs \
    --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

Verify.

```bash
ls -l /var/lib/cockroach/my-safe-directory
```

---

## Step 6. Generate the Node Certificate

**Run As:** `cockroach`

Verify the hostname and IP address.

```bash
hostname -f

hostname -I
```

Generate the node certificate.

```bash
cockroach cert create-node \
  localhost \
  127.0.0.1 \
  <HOSTNAME> \
  <HOST-ONLY-IP> \
  <NAT-IP> \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key

cockroach cert create-node \
  localhost \
  127.0.0.1 \
  oel9-rand  \
  10.10.10.14 \
  192.168.235.240 \
  --certs-dir=/var/lib/cockroach/certs \
  --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

### Parameter Description

| Parameter                            | Description                                           |
| ------------------------------------ | ----------------------------------------------------- |
| localhost                            | Allows local connections using `localhost`.           |
| 127.0.0.1                            | Allows local connections using the loopback address.  |
| `$(hostname -f)`                     | Uses the Fully Qualified Domain Name (FQDN).          |
| `$(hostname -I \| awk '{print $1}')` | Uses the server's primary IP address.                 |
| `--certs-dir`                        | Directory where certificates are stored.              |
| `--ca-key`                           | Path to the CA private key used to sign certificates. |

---

## Step 7. Generate the Root Client Certificate

**Run As:** `cockroach`

```bash
cockroach cert create-client root \
    --certs-dir=/var/lib/cockroach/certs \
    --ca-key=/var/lib/cockroach/my-safe-directory/ca.key
```

---

## Step 8. Verify the Certificates

**Run As:** `cockroach`

```bash
ls -ltr /var/lib/cockroach/certs
```

If logged in as `venkat`, use:

```bash
sudo ls -ltr /var/lib/cockroach/certs
```

Expected files:

```text
ca.crt
node.crt
node.key
client.root.crt
client.root.key
```

---

## Step 9. Configure the Firewall

**Run As:** `venkat`

Open the CockroachDB ports.

```bash
sudo firewall-cmd --permanent --add-port=26257/tcp

sudo firewall-cmd --permanent --add-port=8080/tcp

sudo firewall-cmd --reload
```

Verify.

```bash
sudo firewall-cmd --list-ports
```

| Port  | Purpose                |
| ----- | ---------------------- |
| 26257 | SQL Client Connections |
| 8080  | CockroachDB Admin UI   |

---

## Step 10. Create the Systemd Service

**Run As:** `venkat`

Create the service file.

```bash
sudo vi /etc/systemd/system/cockroach.service
```

Use the following configuration.

> Replace `<SERVER-IP>` with your server's actual IP address or DNS name.

```ini
[Unit]
Description=CockroachDB Database
After=network.target

[Service]
Type=notify
User=cockroach
Group=cockroach

ExecStart=/usr/local/bin/cockroach start-single-node \
    --certs-dir=/var/lib/cockroach/certs \
    --store=/var/lib/cockroach/data \
    --listen-addr=0.0.0.0:26257 \
    --advertise-addr=<SERVER-IP>:26257 \
    --http-addr=0.0.0.0:8080 \
    --cache=25% \
    --max-sql-memory=25%

Restart=always
RestartSec=10

LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

> **Note:** Do not use `$(hostname -I | awk '{print $1}')` in the `ExecStart` command. Systemd does not evaluate shell substitutions. Use a fixed IP address or hostname.

---

## Step 11. Start the CockroachDB Service

**Run As:** `venkat`

```bash
sudo systemctl daemon-reload

sudo systemctl enable cockroach.service

sudo systemctl start cockroach.service
```

Verify.

```bash
sudo systemctl status cockroach -l
```

If the service fails, review the logs.

```bash
sudo journalctl -u cockroach -n 100 --no-pager
```

---

## Step 12. Verify the Listening Ports

**Run As:** `venkat`

```bash
sudo ss -tulpn | grep 26257

sudo ss -tulpn | grep 8080
```

---

## Step 13. Connect Securely to CockroachDB

**Run As:** `cockroach`

```bash
cockroach sql \
    --certs-dir=/var/lib/cockroach/certs \
    --host=<SERVER-IP>
```

Verify the connection.

```sql
SHOW DATABASES;
```

---

## Step 14. Create an Administrative User (Optional)

**Run As:** `cockroach`

```sql
CREATE USER admin WITH PASSWORD 'Training@369';

GRANT admin TO root;
```

---

## Step 15. Validate the Installation

### Access the Admin UI

Open the following URL in your browser.

```text
https://<SERVER-IP>:8080
```

### Verify Cluster Status

```bash
cockroach node status \
    --certs-dir=/var/lib/cockroach/certs \
    --host=<SERVER-IP>:26257
```

Verify the CockroachDB version.

```bash
cockroach sql \
    --certs-dir=/var/lib/cockroach/certs \
    --host=<SERVER-IP> \
    -e "SELECT version();"
```

If all commands complete successfully and the Admin UI is accessible, the CockroachDB installation is complete.
