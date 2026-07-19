### Module 4: Security & User Management

* Authentication (Users, Passwords, Certificates)
* Authorization (Roles, Grants, RBAC)
* TLS (Encryption in Transit)
* Encryption at Rest
* Logging & Audit Logs
* Security Best Practices  

---

### Explore Existing Users

Instead of immediately creating users, first inspect the cluster.

Current user

```sql
SELECT current_user;
```

Current session

```sql
SHOW SESSION_USER;
```

List all users

```sql
SHOW USERS;
```

or

```sql
SELECT username FROM system.users ORDER BY username;
```

Expected (fresh cluster)

```
root
admin
```

---

### Check Existing Roles

Verifing the system roles.

```sql
SHOW ROLES;
```

Example

```
admin
public
```

The `admin` role has administrative privileges, while `public` is granted to every user automatically. Authorization in CockroachDB is based on users, roles, and grants. 

---

### Create Users

```sql
CREATE USER hr_user;

CREATE USER finance_user;

CREATE USER app_user;

CREATE USER auditor;
```

Verify

```sql
SHOW USERS;
```

User details

```sql
SELECT username FROM system.users ORDER BY username;
```

---

### Passwords (Not Password Policies)

CockroachDB supports password authentication, but it does **not** implement Oracle-style password policies such as password expiry, complexity rules, or account lockout. Those controls are typically handled externally (LDAP, OAuth, Kerberos, or organizational policy). 

Create user with password

```sql
CREATE USER appuser WITH PASSWORD 'Welcome@123';
```

Change password

```sql
ALTER USER hr_user WITH PASSWORD 'hruser@123';
```

Remove password

```sql
ALTER USER hr_user WITH PASSWORD NULL;
```

Verify login

```
cockroach sql --host=node1 --user=hr_user
```

---

### Lab 4 – Default Roles

Display roles

```sql
SHOW ROLES;
```

Check role membership

```sql
SHOW ROLE GRANTS;
```

Expected

```
admin

public
```

See who belongs to the admin role

```sql
SHOW GRANTS ON ROLE admin;
```

---

### Lab 5 – Create Custom Roles

```sql
CREATE ROLE hr_role;

CREATE ROLE finance_role;

CREATE ROLE readonly_role;

CREATE ROLE developer_role;
```

Verify

```sql
SHOW ROLES;
```

Assign users

```sql
GRANT hr_role TO hr_user;

GRANT finance_role TO finance_user;

GRANT developer_role TO app_user;
```

Verify

```sql
SHOW ROLE GRANTS;
```

---

### Role Hierarchy

CockroachDB supports nested roles.

Create parent role

```sql
CREATE ROLE manager_role;
```

Grant child roles

```sql
GRANT hr_role TO manager_role;

GRANT finance_role TO manager_role;
```

Create manager

```sql
CREATE USER manager1;
```

Assign

```sql
GRANT manager_role TO manager1;
```

Verify hierarchy

```sql
SHOW ROLE GRANTS;
```

Hierarchy

```
manager1

↓

manager_role

↓

hr_role

↓

finance_role
```

---

### Lab 7 – GRANTS

Grant database

```sql
GRANT CONNECT ON DATABASE security_lab TO hr_role;
```

Grant schema

```sql
GRANT USAGE ON SCHEMA public TO hr_role;
```

Grant table access

```sql
GRANT SELECT ON TABLE company.orders TO hr_user;
```

Grant multiple privileges

```sql
GRANT SELECT,INSERT,UPDATE ON TABLE company.employees TO hr_user;
```

Grant all

```sql
GRANT ALL ON TABLE payroll TO finance_role;
```

Verify

```sql
SHOW GRANTS;
```

Or

```sql
SHOW GRANTS ON TABLE company.employees;
```

---

### Lab 8 – REVOKE

Remove permission

```sql
REVOKE SELECT ON TABLE company.employees FROM hr_role;
```

Check

```sql
SHOW GRANTS ON TABLE employees;
```

Reconnect as HR user and try

```
cockroach sql --user=hr_user --insecure
```

Execute

```sql
SELECT * FROM company.employees;
```

Expected

```
ERROR:

permission denied
```

---

### RBAC

Instead of granting privileges to every user

```
User1

User2

User3

User4
```

Create one role

```sql
CREATE ROLE application_role;
```

Grant privileges

```sql
GRANT SELECT,INSERT,UPDATE ON company.employees TO application_role;
```

Assign users

```sql
CREATE USER appuser1;

CREATE USER appuser2;

CREATE USER appuser3;

GRANT application_role TO appuser1;

GRANT application_role TO appuser2;

GRANT application_role TO appuser3;
```

Now every member inherits the same permissions. This role-based model is the recommended authorization approach in CockroachDB. 

---

### Lab 10 – Encryption in Transit (TLS)

Check certificates

```bash
ls certs/
```

```
ca.crt
ca.key

node.crt
node.key

client.root.crt
client.root.key
```

Secure connection

```bash
cockroach sql --host=node1 --certs-dir=certs
```

Verify TLS

```bash
openssl s_client -connect node1:26257
```

Observe

```
TLSv1.3

Cipher

Certificate

Handshake
```

The security chapter recommends encrypting client/server communication with TLS certificates. 

---

### Encryption at Rest

Encryption at Rest is an **Enterprise** feature.

Check cluster settings

```sql
SHOW CLUSTER SETTINGS;
```

Search encryption settings

```sql
SHOW CLUSTER SETTINGS LIKE '%encryption%';
```

Here CockroachDB encrypts on-disk data using data encryption keys protected by store/master keys, protecting data if disks are compromised. 

---

### Audit Logs

CockroachDB supports SQL logging and fine-grained audit logging for sensitive access. 

Enable auditing (Enterprise)

```sql
ALTER TABLE employees SET EXPERIMENTAL_AUDIT='READ WRITE';
```

To Find Enterprise Edition or Not ? Output Should not Empty if it is Enterprise Edition
```
SHOW CLUSTER SETTING enterprise.license;
```

```
 SHOW CLUSTER SETTING enterprise.license;
```

Generate activity

```sql
SELECT * FROM employees;

UPDATE employees SET salary=90000 WHERE emp_id=1;

DELETE FROM employees WHERE emp_id=2;
```

Inspect logs

```bash
cd logs
```

```bash
tail -f cockroach.log
```

You can also inspect the SQL execution log and sensitive access log if configured:

```bash
grep SQL_EXEC logs/cockroach.log
```

```bash
grep SENSITIVE_ACCESS logs/cockroach.log
```


