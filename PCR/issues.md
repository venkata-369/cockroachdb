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

> ⚠️ Please share the output of `sudo journalctl -xeu cockroach.service | tail -50` so I can give you the exact fix based on the actual error message.
