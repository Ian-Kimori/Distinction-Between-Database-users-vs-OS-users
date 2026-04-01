# Distinction-Database-users-vs-OS-users

---

**Database users vs OS users — they are completely separate:**

---

**OS users** — defined in `/etc/passwd`
```bash
# These are OS users
cat /etc/passwd | grep -v nologin
```

| OS User | Purpose |
|---|---|
| `root` | OS superuser |
| `mysql` | Service account that runs the MariaDB process |
| `leo_audit` | Your audit login account |
| `ubuntu` | System user |

OS users are used to:
- Log into the Linux shell via SSH
- Run OS commands
- Own files and processes
- The `mysql` OS user owns the MariaDB process but **cannot log into MariaDB itself**

---

**Database users** — defined inside MariaDB in `mysql.user`
```sql
-- These are DB users
SELECT user, host FROM mysql.user;
```

| DB User | Purpose |
|---|---|
| `root` | MariaDB superuser — completely separate from OS root |
| `galera_sst` | Galera cluster replication |
| `backup_user` | Backup operations |
| `app_payments` | Application account |
| `zabbix` | Monitoring |

DB users are used to:
- Authenticate to the MariaDB server
- Execute SQL queries
- Access databases and tables
- They exist **only inside MariaDB** — they have no OS shell access

---

**Key differences:**

| Aspect | OS User | DB User |
|---|---|---|
| Defined in | `/etc/passwd` | `mysql.user` table |
| Managed with | `useradd`, `usermod` | `CREATE USER`, `ALTER USER` |
| Authenticates to | Linux shell / SSH | MariaDB server |
| Password stored in | `/etc/shadow` | `mysql.user.authentication_string` |
| Has shell access | Yes (if configured) | No |
| Has DB access | Not automatically | Yes |
| `root` means | Full OS control | Full DB control |

---

**Important — OS root ≠ DB root:**

| Account | Access |
|---|---|
| OS `root` | Can read MariaDB data files directly on disk but cannot run SQL unless also has DB credentials |
| DB `root` | Full SQL privileges inside MariaDB but no OS shell access |
| Both `root` accounts can exist independently with different passwords | |

---

**For your X509 test specifically:**

You use a **database username** — one of the users returned by:
```sql
SELECT user, host, ssl_type 
FROM mysql.user 
WHERE user NOT IN ('mysql','root','mariadb.sys')
AND ssl_type = 'X509';
```

Then the connection command uses:
```bash
# 'appuser' here is the DATABASE user, not an OS user
mysql -u appuser -p -h <db-server-ip> \
  --ssl-ca=/etc/my.cnf.d/certs/ca.crt \
  --ssl-mode=VERIFY_CA \
  -e "SELECT CURRENT_USER();"
```

---

**The only exception — `unix_socket` authentication:**

This is the one case where OS and DB users are linked:
```sql
-- If a DB user uses unix_socket plugin
SELECT user, plugin FROM mysql.user WHERE plugin = 'unix_socket';
```

With `unix_socket` authentication:
- The DB user must have the **same name** as the OS user
- Authentication happens via the OS socket — no password needed
- Example: OS user `leo_audit` can connect as DB user `leo_audit` without a password if that DB account uses `unix_socket`

| Condition | Meaning |
|---|---|
| DB user plugin = `unix_socket` | OS username must match DB username |
| DB user plugin = `mysql_native_password` or `ed25519` | Completely independent of OS users |
