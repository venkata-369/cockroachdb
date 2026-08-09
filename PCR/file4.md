### ⏭️ PCR - Physical Cluster Replication Setup: 

### Architecture Reminder

| Cluster | Region | Role |
|---|---|---|
| Singapore (crdb-node8, crdb-node9) | ap-southeast-1 | **Primary** |
| Mumbai (crdb-node3, crdb-node4) | ap-south-1 | **Standby** |

---

## Step 1: Exchange CA Certificates Between Clusters

PCR requires each cluster to trust the other's CA certificate. [[PCR Setup](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication#step-3-manage-cluster-certificates-and-generate-connection-strings)]

### Download Singapore CA to your local machine:

```powershell
scp -i ~/.ssh/id_rsa ubuntu@54.179.159.35:/home/ubuntu/certs/ca.crt ~/sg-ca.crt
```

### Upload Singapore CA to Mumbai nodes:

```powershell
scp -i ~/.ssh/id_rsa ~/sg-ca.crt ubuntu@<Mumbai-Node3-Public-IP>:/home/ubuntu/certs/ca-singapore.crt
scp -i ~/.ssh/id_rsa ~/sg-ca.crt ubuntu@<Mumbai-Node4-Public-IP>:/home/ubuntu/certs/ca-singapore.crt
```

### On **both Mumbai nodes** (crdb-node3 and crdb-node4), copy Singapore CA to cockroach certs:

```bash
sudo cp /home/ubuntu/certs/ca-singapore.crt /var/lib/cockroach/certs/ca-singapore.crt
sudo chown cockroach:cockroach /var/lib/cockroach/certs/ca-singapore.crt
sudo chmod 644 /var/lib/cockroach/certs/ca-singapore.crt
```

---

## Step 2: Create Replication User on Singapore (Primary)

On **crdb-node8** (Singapore):

```bash
cockroach sql \
  --certs-dir=/home/ubuntu/certs \
  --host=10.30.2.151
```

Then in the SQL shell:

```sql
CREATE USER replicator WITH PASSWORD 'repl123';
GRANT SYSTEM REPLICATION TO replicator;
```

Type `\q` to exit.

---

## Step 3: Generate Encoded Connection String on Singapore

On **crdb-node8** (Singapore): [[PCR Connection String](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication#step-3-manage-cluster-certificates-and-generate-connection-strings)]

```bash
cockroach encode-uri \
  replicator:repl123@10.30.2.151:26257 \
  --ca-cert /home/ubuntu/certs/ca.crt \
  --inline
```

This outputs a connection string like:

```
ubuntu@crdb-node8:~$ cockroach encode-uri \
  replicator:repl123@10.30.2.151:26257 \
  --ca-cert /home/ubuntu/certs/ca.crt \
  --inline
replicator:repl123@10.30.2.151:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJTCCAg2gAwIBAgIQXUsgZp4IqL%2FrFNXvZGnWBTANBgkqhkiG9w0BAQsFADAr%0AMRIwEAYDVQQKEwlDb2Nrcm9hY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTAeFw0y%0ANjA4MDgwMjAzMDJaFw0zNjA4MTYwMjAzMDJaMCsxEjAQBgNVBAoTCUNvY2tyb2Fj%0AaDEVMBMGA1UEAxMMQ29ja3JvYWNoIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A%0AMIIBCgKCAQEArXCI9ryYMZeuugQjIJdncUSNhFYP%2F7jjiFMD0G%2Fxw%2BukapfaypYy%0Ae9Y410jvHM6%2FcV%2F2hORYwOm8Gc1WvlkDRI3vQv8rO3N%2BBHxxUPiRfmvu16haS%2By7%0AlpQHFHJV%2B61wtvMPU695lW7LC2Q5SSYMjd8G%2BoN4tmspd7GotRChPMp3pPT16Yon%0AWCkb0GcgnUyGhoPjUrRqEygMlfL6YXVmtnyGhdoXfZ2rI0035SHucj8UZI2X4ulX%0AUniltklIPWML%2FPQapMIjPN75YtFXtQCVPBQFIB0xUitev6s2Qa32xWRWg35uHIin%0Aue9pK41qi5ddLkl3mCzGRe%2FZhwkAP6wqywIDAQABo0UwQzAOBgNVHQ8BAf8EBAMC%0AAuQwEgYDVR0TAQH%2FBAgwBgEB%2FwIBATAdBgNVHQ4EFgQU0ChaFMBxxWS7haPaypVj%0AL726nu4wDQYJKoZIhvcNAQELBQADggEBAGcej8RLRvFgaItlEJPrfDrHEnLg3v33%0AMgHzAMeldsI0a5uFZamDo1oFNAERlaedcs7WWPmwVQ%2BOcFtYwD58bJLKraZkW2Fv%0A2TWu4N5dWOVVg3ZLRL49GNpoQq95p%2BAUZA%2FzsM5kfwfuQ1NzQwBa%2FwrN0JGCUpah%0AKWCqAtjPH%2FxrzLSEYSTgoknMu%2F80tIJ1wzip5V%2Fr97xbgMTa2TqxzpvsF7EnVYgo%0AUtRoA%2FB8h%2FhWB99BPosmiGGXg%2FPnCpJe7KnJUTW0IKYcNe%2BSlPw3tuqyn56DJe9G%0A4Qs3yOI2sgv%2BMQ5tWc4fAeI%2F5%2FKGdFUJ9rdlgTYLN%2FMuqpe%2BQh2ZYDI%3D%0A-----END+CERTIFICATE-----%0A
```

**Copy this output** — you will need it in Step 4.

```
replicator:repl123@10.30.2.151:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJTCCAg2gAwIBAgIQXUsgZp4IqL%2FrFNXvZGnWBTANBgkqhkiG9w0BAQsFADAr%0AMRIwEAYDVQQKEwlDb2Nrcm9hY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTAeFw0y%0ANjA4MDgwMjAzMDJaFw0zNjA4MTYwMjAzMDJaMCsxEjAQBgNVBAoTCUNvY2tyb2Fj%0AaDEVMBMGA1UEAxMMQ29ja3JvYWNoIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A%0AMIIBCgKCAQEArXCI9ryYMZeuugQjIJdncUSNhFYP%2F7jjiFMD0G%2Fxw%2BukapfaypYy%0Ae9Y410jvHM6%2FcV%2F2hORYwOm8Gc1WvlkDRI3vQv8rO3N%2BBHxxUPiRfmvu16haS%2By7%0AlpQHFHJV%2B61wtvMPU695lW7LC2Q5SSYMjd8G%2BoN4tmspd7GotRChPMp3pPT16Yon%0AWCkb0GcgnUyGhoPjUrRqEygMlfL6YXVmtnyGhdoXfZ2rI0035SHucj8UZI2X4ulX%0AUniltklIPWML%2FPQapMIjPN75YtFXtQCVPBQFIB0xUitev6s2Qa32xWRWg35uHIin%0Aue9pK41qi5ddLkl3mCzGRe%2FZhwkAP6wqywIDAQABo0UwQzAOBgNVHQ8BAf8EBAMC%0AAuQwEgYDVR0TAQH%2FBAgwBgEB%2FwIBATAdBgNVHQ4EFgQU0ChaFMBxxWS7haPaypVj%0AL726nu4wDQYJKoZIhvcNAQELBQADggEBAGcej8RLRvFgaItlEJPrfDrHEnLg3v33%0AMgHzAMeldsI0a5uFZamDo1oFNAERlaedcs7WWPmwVQ%2BOcFtYwD58bJLKraZkW2Fv%0A2TWu4N5dWOVVg3ZLRL49GNpoQq95p%2BAUZA%2FzsM5kfwfuQ1NzQwBa%2FwrN0JGCUpah%0AKWCqAtjPH%2FxrzLSEYSTgoknMu%2F80tIJ1wzip5V%2Fr97xbgMTa2TqxzpvsF7EnVYgo%0AUtRoA%2FB8h%2FhWB99BPosmiGGXg%2FPnCpJe7KnJUTW0IKYcNe%2BSlPw3tuqyn56DJe9G%0A4Qs3yOI2sgv%2BMQ5tWc4fAeI%2F5%2FKGdFUJ9rdlgTYLN%2FMuqpe%2BQh2ZYDI%3D%0A-----END+CERTIFICATE-----%0A
```
---

## Step 4: Start Replication from Mumbai (Standby)

SSH into **crdb-node3** (Mumbai) and open SQL shell:

```bash
cockroach sql \
  --certs-dir=/home/ubuntu/certs \
  --host=10.10.3.10
```

Then run:

```sql
CREATE VIRTUAL CLUSTER main
  FROM REPLICATION OF system
  ON 'postgresql://replicator:repl123@10.30.2.151:26257?options=-ccluster%3Dsystem&sslinline=true&sslmode=verify-full&sslrootcert=-----BEGIN+CERTIFICATE-----%0AMIIDJTCCAg2gAwIBAgIQXUsgZp4IqL%2FrFNXvZGnWBTANBgkqhkiG9w0BAQsFADAr%0AMRIwEAYDVQQKEwlDb2Nrcm9hY2gxFTATBgNVBAMTDENvY2tyb2FjaCBDQTAeFw0y%0ANjA4MDgwMjAzMDJaFw0zNjA4MTYwMjAzMDJaMCsxEjAQBgNVBAoTCUNvY2tyb2Fj%0AaDEVMBMGA1UEAxMMQ29ja3JvYWNoIENBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A%0AMIIBCgKCAQEArXCI9ryYMZeuugQjIJdncUSNhFYP%2F7jjiFMD0G%2Fxw%2BukapfaypYy%0Ae9Y410jvHM6%2FcV%2F2hORYwOm8Gc1WvlkDRI3vQv8rO3N%2BBHxxUPiRfmvu16haS%2By7%0AlpQHFHJV%2B61wtvMPU695lW7LC2Q5SSYMjd8G%2BoN4tmspd7GotRChPMp3pPT16Yon%0AWCkb0GcgnUyGhoPjUrRqEygMlfL6YXVmtnyGhdoXfZ2rI0035SHucj8UZI2X4ulX%0AUniltklIPWML%2FPQapMIjPN75YtFXtQCVPBQFIB0xUitev6s2Qa32xWRWg35uHIin%0Aue9pK41qi5ddLkl3mCzGRe%2FZhwkAP6wqywIDAQABo0UwQzAOBgNVHQ8BAf8EBAMC%0AAuQwEgYDVR0TAQH%2FBAgwBgEB%2FwIBATAdBgNVHQ4EFgQU0ChaFMBxxWS7haPaypVj%0AL726nu4wDQYJKoZIhvcNAQELBQADggEBAGcej8RLRvFgaItlEJPrfDrHEnLg3v33%0AMgHzAMeldsI0a5uFZamDo1oFNAERlaedcs7WWPmwVQ%2BOcFtYwD58bJLKraZkW2Fv%0A2TWu4N5dWOVVg3ZLRL49GNpoQq95p%2BAUZA%2FzsM5kfwfuQ1NzQwBa%2FwrN0JGCUpah%0AKWCqAtjPH%2FxrzLSEYSTgoknMu%2F80tIJ1wzip5V%2Fr97xbgMTa2TqxzpvsF7EnVYgo%0AUtRoA%2FB8h%2FhWB99BPosmiGGXg%2FPnCpJe7KnJUTW0IKYcNe%2BSlPw3tuqyn56DJe9G%0A4Qs3yOI2sgv%2BMQ5tWc4fAeI%2F5%2FKGdFUJ9rdlgTYLN%2FMuqpe%2BQh2ZYDI%3D%0A-----END+CERTIFICATE-----%0A';
```

---

## Step 5: Verify Replication is Running on Mumbai

```sql
SHOW VIRTUAL CLUSTERS;
```

Expected output:

```
  id |  name  |        data_state        | service_mode
-----+--------+--------------------------+---------------
   1 | system | ready                    | shared
   3 | main   | initializing replication | none
```

[[PCR Example](https://www.cockroachlabs.com/docs/v25.2/set-up-physical-cluster-replication#example)]

WorkLog Output:-
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
CREATE VIRTUAL CLUSTER FROM REPLICATION 0

Time: 631ms total (execution 631ms / network 0ms)

root@10.10.3.10:26257/defaultdb> SHOW VIRTUAL CLUSTERS;
  id |  name  |      data_state      | service_mode
-----+--------+----------------------+---------------
   1 | system | ready                | shared
   3 | main   | running initial scan | none
(2 rows)

Time: 10ms total (execution 9ms / network 0ms)
```
---

#### ⚠️ Important Notes

| Item | Detail |
|---|---|
| Run `CREATE VIRTUAL CLUSTER` | Only on **Mumbai (standby)** |
| Connection string | Points to **Singapore (primary)** |
| `initializing replication` state | Normal — replication is starting |
| CA exchange | Required for clusters to trust each other |

## ✅ PCR Replication Stream Started Successfully!

The output confirms replication is working:

| id | name | data_state | service_mode |
|---|---|---|---|
| 1 | system | ready | shared |
| 3 | main | **running initial scan** | none |

`running initial scan` means Mumbai is currently copying all existing data from Singapore. This is completely normal! [[PCR Monitoring CockroachDB Documentation](https://www.cockroachlabs.com/docs/v25.2/physical-cluster-replication-monitoring#sql-shell)]

---

### ⏭️ Monitor Replication Progress

Run this on **crdb-node3** (Mumbai) to watch the replication status:

```sql
SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
```

### States You Will See in Order:

| State | Meaning |
|---|---|
| `running initial scan` | Copying existing data from Singapore |
| `initializing replication` | Initial scan done, setting up stream |
| `replicating` | ✅ Fully replicating — steady state |

---

## Keep Monitoring Until `replicating` State

```sql
SHOW VIRTUAL CLUSTER main WITH REPLICATION STATUS;
```

Expected final output:
```
  id | name | source_tenant_name | replicated_time | replication_lag | status
-----+------+--------------------+-----------------+-----------------+-------------
   3 | main | system             | 2026-08-09 ...  | 00:00:XX        | replicating
```

Key fields to check:
- `replication_lag` — should be small (seconds)
- `replicated_time` — should be advancing every ~30s
- `status` — should show `replicating`

[[PCR Monitoring](https://www.cockroachlabs.com/docs/v25.2/physical-cluster-replication-monitoring#sql-shell)]

---

#### ✅ Overall PCR Status

| Item | Status |
|---|---|
| Singapore (Primary) | ✅ Running |
| Mumbai (Standby) | ✅ Running |
| Replication Stream | ✅ Started |
| Initial Scan | ⏳ In Progress |


