## Step 1: Check the Error Details

```bash
sudo systemctl status cockroach.service
```
---
```
ubuntu@crdb-node4:~$ sudo systemctl status cockroach.service
● cockroach.service - CockroachDB Database
     Loaded: loaded (/etc/systemd/system/cockroach.service; enabled; preset: enabled)
     Active: activating (auto-restart) (Result: exit-code) since Sun 2026-08-09 01:01:11 UTC; 2s ago
    Process: 11506 ExecStart=/usr/local/bin/cockroach start --certs-dir=${CERTS_DIR} --store=${DATA_DIR} --listen-addr=0.0.0.0:26257 --advertise-addr=${NODE>
   Main PID: 11506 (code=exited, status=1/FAILURE)
        CPU: 737ms
```
```
ubuntu@crdb-node4:~$ sudo journalctl -xeu cockroach.service | tail -50
░░ The job identifier is 25201 and the job result is failed.
Aug 09 01:01:28 crdb-node4 systemd[1]: cockroach.service: Scheduled restart job, restart counter is at 15.
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ Automatic restarting of the unit cockroach.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
Aug 09 01:01:28 crdb-node4 systemd[1]: Starting cockroach.service - CockroachDB Database...
░░ Subject: A start job for unit cockroach.service has begun execution
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit cockroach.service has begun execution.
░░
░░ The job identifier is 25317.
Aug 09 01:01:29 crdb-node4 cockroach[11547]: initiating hard shutdown of server
Aug 09 01:01:29 crdb-node4 cockroach[11547]: too early to drain; used hard shutdown instead
Aug 09 01:01:29 crdb-node4 cockroach[11547]: *
Aug 09 01:01:29 crdb-node4 cockroach[11547]: * ERROR: ERROR: cannot load certificates.
Aug 09 01:01:29 crdb-node4 cockroach[11547]: * Check your certificate settings, set --certs-dir, or use --insecure for insecure clusters.
Aug 09 01:01:29 crdb-node4 cockroach[11547]: *
Aug 09 01:01:29 crdb-node4 cockroach[11547]: * server startup failed: failed to start server: problem using security settings: problem loading certs directory /var/lib/cockroach/certs: open /var/lib/cockroach/certs: permission denied
Aug 09 01:01:29 crdb-node4 cockroach[11547]: *
Aug 09 01:01:29 crdb-node4 cockroach[11547]: ERROR: cannot load certificates.
Aug 09 01:01:29 crdb-node4 cockroach[11547]: Check your certificate settings, set --certs-dir, or use --insecure for insecure clusters.
Aug 09 01:01:29 crdb-node4 cockroach[11547]: server startup failed: failed to start server: problem using security settings: problem loading certs directory /var/lib/cockroach/certs: open /var/lib/cockroach/certs: permission denied
Aug 09 01:01:29 crdb-node4 cockroach[11547]: Failed running "start"
Aug 09 01:01:29 crdb-node4 systemd[1]: cockroach.service: Main process exited, code=exited, status=1/FAILURE
░░ Subject: Unit process exited
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ An ExecStart= process belonging to unit cockroach.service has exited.
░░
░░ The process' exit code is 'exited' and its exit status is 1.
Aug 09 01:01:29 crdb-node4 systemd[1]: cockroach.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ The unit cockroach.service has entered the 'failed' state with result 'exit-code'.
Aug 09 01:01:29 crdb-node4 systemd[1]: Failed to start cockroach.service - CockroachDB Database.
░░ Subject: A start job for unit cockroach.service has failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit cockroach.service has finished with a failure.
░░
░░ The job identifier is 25317 and the job result is failed.

```
---
```bash
sudo journalctl -xeu cockroach.service | tail -50
```

---

## Step 2: Most Likely Causes and Fixes

### Cause 1: Service file not saved correctly

Verify the service file has no `--insecure`:

```bash
systemctl cat cockroach
```

Look for `--certs-dir` in `ExecStart` and confirm **no** `--insecure` flag.

---

### Cause 2: Log directory does not exist

```bash
sudo mkdir -p /var/lib/cockroach/logs
sudo chown -R cockroach:cockroach /var/lib/cockroach/logs
```

---

### Cause 3: Data directory permissions wrong

```bash
sudo mkdir -p /var/lib/cockroach/data
sudo chown -R cockroach:cockroach /var/lib/cockroach/data
sudo chmod 700 /var/lib/cockroach/data
```

---

### Cause 4: Certs directory permissions wrong

```bash
sudo chown -R cockroach:cockroach /var/lib/cockroach/certs
sudo chmod 700 /var/lib/cockroach/certs
sudo chmod 600 /var/lib/cockroach/certs/node.key
sudo chmod 644 /var/lib/cockroach/certs/ca.crt
sudo chmod 644 /var/lib/cockroach/certs/node.crt
```

---

## Step 3: After Fixes, Restart

```bash
sudo systemctl daemon-reload
sudo systemctl start cockroach
sudo systemctl status cockroach
```

[[Deploy On-Premises](https://www.cockroachlabs.com/docs/stable/deploy-cockroachdb-on-premises#step-3-start-nodes)]

---
> ⚠️ Please share the output of `sudo journalctl -xeu cockroach.service | tail -50` so I can give you the exact fix based on the actual error message
---
## ✅ Node 4 is Now Running in Secure Mode!

The CGroup line confirms it:
```
cockroach start --certs-dir=/var/lib/cockroach/certs ...
```

The warning `unable to contact the other nodes` is **completely normal** at this stage — it just means the cluster has not been initialized yet. Both nodes are waiting for `cockroach init`. [[Cluster Node Availability](https://www.cockroachlabs.com/docs/v25.2/cluster-setup-troubleshooting#client-connection-issues)]

---

### ✅ Current Status

| Node | Mode | Status |
|---|---|---|
| Node 3 (10.10.3.10) | ✅ Secure | ✅ Running |
| Node 4 (10.10.4.10) | ✅ Secure | ✅ Running |

---

### ⏭️ Next Step: Initialize the Cluster

Run this **once only** from **Node 3**:

```bash
cockroach init --certs-dir=/home/ubuntu/certs --host=10.10.3.10
```


You should see:
```
Cluster successfully initialized
```

---

## Then Verify Both Nodes are Live

```bash
cockroach node list \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10
```

Expected output:
```
  id |     address      | is_available | is_live
-----+------------------+--------------+---------
   1 | 10.10.3.10:26257 | true         | true
   2 | 10.10.4.10:26257 | true         | true
```
Start Replication from Mumbai (Standby) --- Singapore (Primary)
```
root@10.10.3.10:26257/defaultdb> CREATE VIRTUAL CLUSTER main
                              ->   FROM REPLICATION OF system
                              ->   ON
                              -> 'postgresql://replicator:repl123@10.30.2.151:26257/defaultdb?options=-ccluster%3Dsystem&sslinline=true&sslmode
                              -> =verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJTCCAg2gAwIBAgIQXUsgZp4IqL%2FrFNXvZGnWBTANBgkqhkiG9
                              -> w0BAQsFADAr%0AMRIwEAYDV
                              -> QQKEwlDb2Nrcm9hY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTAeFw0y%0ANjA4MDgwMjAzMDJaFw0zNjA4MTYwMjAzMDJaMCsxEjAQBgNVBAoT
                              ->
                              -> CUNvY2tyb2Fj%0AaDEVMBMGA1UEAxMMQ29ja3JvYWNoIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A...-----END+CERTIFICATE-----%0A
                              -> ';
                              ->
ERROR: unable to add CA to cert pool
```
```
root@10.10.3.10:26257/defaultdb> CREATE VIRTUAL CLUSTER main
                              ->   FROM REPLICATION OF system
                              ->   ON
                              -> 'postgresql://replicator:repl123@10.30.2.151:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-fu
                              -> ll&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJTCCAg2gAwIBAgIQXUsgZp4IqL%2FrFNXvZGnWBTANBgkqhkiG9w0BAQsFADA
                              -> r%0AMRIwEAYDVQQKEwlDb2Nrcm9hY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTAeFw0y%0ANjA4MDgwMjAzMDJaFw0zNjA4MTYwMjAzMDJaMCs
                              -> xEjAQBgNVBAoTCUNvY2tyb2Fj%0AaDEVMBMGA1UEAxMMQ29ja3JvYWNoIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A%0AMIIBCgKCAQEArXC
                              -> I9ryYMZeuugQjIJdncUSNhFYP%2F7jjiFMD0G%2Fxw%2BukapfaypYy%0Ae9Y410jvHM6%2FcV%2F2hORYwOm8Gc1WvlkDRI3vQv8rO3N%2BBH
                              -> xxUPiRfmvu16haS%2By7%0AlpQHFHJV%2B61wtvMPU695lW7LC2Q5SSYMjd8G%2BoN4tmspd7GotRChPMp3pPT16Yon%0AWCkb0GcgnUyGhoPj
                              -> UrRqEygMlfL6YXVmtnyGhdoXfZ2rI0035SHucj8UZI2X4ulX%0AUniltklIPWML%2FPQapMIjPN75YtFXtQCVPBQFIB0xUitev6s2Qa32xWRWg
                              -> 35uHIin%0Aue9pK41qi5ddLkl3mCzGRe%2FZhwkAP6wqywIDAQABo0UwQzAOBgNVHQ8BAf8EBAMC%0AAuQwEgYDVR0TAQH%2FBAgwBgEB%2FwI
                              -> BATAdBgNVHQ4EFgQU0ChaFMBxxWS7haPaypVj%0AL726nu4wDQYJKoZIhvcNAQELBQADggEBAGcej8RLRvFgaItlEJPrfDrHEnLg3v33%0AMgH
                              -> zAMeldsI0a5uFZamDo1oFNAERlaedcs7WWPmwVQ%2BOcFtYwD58bJLKraZkW2Fv%0A2TWu4N5dWOVVg3ZLRL49GNpoQq95p%2BAUZA%2FzsM5k
                              -> fwfuQ1NzQwBa%2FwrN0JGCUpah%0AKWCqAtjPH%2FxrzLSEYSTgoknMu%2F80tIJ1wzip5V%2Fr97xbgMTa2TqxzpvsF7EnVYgo%0AUtRoA%2F
                              -> B8h%2FhWB99BPosmiGGXg%2FPnCpJe7KnJUTW0IKYcNe%2BSlPw3tuqyn56DJe9G%0A4Qs3yOI2sgv%2BMQ5tWc4fAeI%2F5%2FKGdFUJ9rdlg
                              -> TYLN%2FMuqpe%2BQh2ZYDI%3D%0A-----END+CERTIFICATE-----%0A';
ERROR: error creating replication stream for tenant system: ERROR: crdb_internal.start_replication_stream(): kv.rangefeed.enabled must be true to start a replication job (SQLSTATE XXUUU)
root@10.10.3.10:26257/defaultdb>
```
### Notes to Fix:-
## Fix: Enable `kv.rangefeed.enabled` on Mumbai (Standby)

The error is clear:
```
kv.rangefeed.enabled must be true to start a replication job
```

You need to enable rangefeeds on the **Mumbai standby cluster's system virtual cluster**. [[PCR Primary Cluster Setup](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication#step-1-create-the-primary-cluster)]

---

## Fix: Connect to Mumbai System Virtual Cluster

On **crdb-node3** (Mumbai), exit the current SQL shell first:

```sql
\q
```
Then reconnect specifically to the **system virtual cluster**:

```bash
cockroach sql \
  --url "postgresql://root@10.10.3.10:26257?options=-ccluster=system&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

---

## Enable Rangefeed on Mumbai System Virtual Cluster

```sql
SET CLUSTER SETTING kv.rangefeed.enabled = true;
```

Verify:

```sql
SHOW CLUSTER SETTING kv.rangefeed.enabled;
```

Expected:
```
  kv.rangefeed.enabled
------------------------
        true
```
---
WorkLog Output 
```
ubuntu@crdb-node3:~$ cockroach sql \
  --url "postgresql://root@10.10.3.10:26257?options=-ccluster=system&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
#
# Welcome to the CockroachDB SQL shell.
# All statements must be terminated by a semicolon.
# To exit, type: \q.
#
# Server version: CockroachDB CCL v25.2.2 (x86_64-pc-linux-gnu, built 2025/06/23 13:45:25, go1.23.7 X:nocoverageredesign) (same version as client)
# Cluster ID: 4301c167-821d-43de-bd5d-2d43ed3ecb44
# Organization: djs-colo
#
# Enter \? for a brief introduction.
#
root@10.10.3.10:26257/defaultdb> SET CLUSTER SETTING kv.rangefeed.enabled = true;
SET CLUSTER SETTING

Time: 4ms total (execution 4ms / network 0ms)

root@10.10.3.10:26257/defaultdb> SHOW CLUSTER SETTING kv.rangefeed.enabled;
  kv.rangefeed.enabled
------------------------
           t
(1 row)

Time: 1ms total (execution 0ms / network 0ms)
```
---

## Also Enable on Singapore (Primary) System Virtual Cluster

On **crdb-node8** (Singapore):

```bash
cockroach sql \
  --url "postgresql://root@10.30.2.151:26257?options=-ccluster=system&sslmode=verify-full" \
  --certs-dir=/home/ubuntu/certs
```

```sql
SET CLUSTER SETTING kv.rangefeed.enabled = true;
```

---

#### Then Retry `CREATE VIRTUAL CLUSTER` on Mumbai

```sql
CREATE VIRTUAL CLUSTER main
  FROM REPLICATION OF system
  ON 'postgresql://replicator:repl123@10.30.2.151:26257/defaultdb?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----...-----END+CERTIFICATE-----%0A';
```

[[PCR Example](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication#example)]
