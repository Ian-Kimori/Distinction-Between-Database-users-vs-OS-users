# Distinction-Between-Database-users-vs-OS-users

---

# Part 1 — Types of OS Accounts

OS accounts are defined in `/etc/passwd` and managed by the Linux operating system.

---

## OS Account Types

**1. Root account**
```bash
root:x:0:0:root:/root:/bin/bash
```
| Property | Value |
|---|---|
| Purpose | Full OS superuser — unrestricted access to everything |
| Has shell | ✅ Yes — `/bin/bash` |
| Authenticates via | Password or SSH key |
| DB access | Only if DB credentials also configured |

---

**2. System service accounts**
```bash
mysql:x:27:27:MariaDB Server:/var/lib/mysql:/sbin/nologin
zabbix:x:997:995:Zabbix:/var/lib/zabbix:/sbin/nologin
```
| Property | Value |
|---|---|
| Purpose | Run a specific service process — MariaDB, Zabbix, nginx etc |
| Has shell | ❌ No — `/sbin/nologin` prevents interactive login |
| Authenticates via | Not applicable — cannot log in |
| DB access | Not automatically — needs separate DB account |
| How created | Automatically by package installer |

---

**3. Human/interactive accounts**
```bash
leo_audit:x:1004:1004::/home/leo_audit:/bin/bash
ubuntu:x:1000:1000::/home/ubuntu:/bin/bash
```
| Property | Value |
|---|---|
| Purpose | Named person with interactive shell access |
| Has shell | ✅ Yes — `/bin/bash` |
| Authenticates via | Password or SSH key |
| DB access | Only if DB credentials also configured |

---

**4. Application service accounts**

Created specifically for an application to run as:
```bash
appuser:x:1005:1005::/home/appuser:/sbin/nologin
```
| Property | Value |
|---|---|
| Purpose | Run application processes under a non-root identity |
| Has shell | ❌ Usually no |
| Authenticates via | Not applicable for login |
| DB access | Via DB account configured for that application |

---

**How OS accounts authenticate:**

| Method | How it works |
|---|---|
| Password | Stored hashed in `/etc/shadow` |
| SSH key pair | Public key in `~/.ssh/authorized_keys` |
| SSSD + Active Directory | Kerberos ticket via your AD domain (your setup) |
| PAM | Pluggable Authentication Modules — can chain multiple methods |

---

# Part 2 — Types of Database Accounts

DB accounts are defined inside MariaDB in the `mysql.user` table. They are completely separate from OS accounts.

---

## DB Account Types

**1. Root DB account**
```sql
'root'@'localhost'
```
| Property | Value |
|---|---|
| Purpose | Full MariaDB superuser — all privileges on everything |
| Is a process | ❌ No — it is a login account |
| Authenticates via | Password or `unix_socket` plugin |
| Shell access | ❌ No — DB accounts never have shell access |
| Equivalent OS role | Similar to OS root but only inside MariaDB |

---

**2. System reserved accounts**
```sql
'mysql.sys'@'localhost'
'mysql.session'@'localhost'
'mysql.infoschema'@'localhost'
'mariadb.sys'@'localhost'
```
| Property | Value |
|---|---|
| Purpose | Internal MariaDB system operations |
| Is a process | ❌ No — they are accounts used internally by MariaDB engine |
| Should be locked | ✅ Yes — `account_locked = true` |
| Shell access | ❌ No |
| Can log in | ❌ No — locked by design |

---

**3. Galera SST account (`sstuser`)**
```sql
'sstuser'@'localhost'
'sstuser'@'10.19.6.96'
```
| Property | Value |
|---|---|
| Purpose | Galera cluster State Snapshot Transfer — syncing a new or rejoining node |
| Is a process | ❌ No — it is a DB login account used by the Galera process |
| Used by | The `wsrep` Galera process running on each node |
| Shell access | ❌ No |
| Required privileges | `RELOAD`, `LOCK TABLES`, `REPLICATION CLIENT`, `REPLICATION SLAVE` |
| Authenticates via | Password stored in wsrep config |

**What SST actually does:**

When a Galera node goes down and rejoins the cluster it needs to receive a full or incremental copy of the database from another node. The `sstuser` account is what the receiving node uses to authenticate to the donor node and pull that data. It is used by the Galera software process — not by a human.

```
Node A (donor)          Node B (rejoining)
     |                        |
     |<-- sstuser connects ----|
     |--- sends full snapshot ->|
     |                        |
     | wsrep process uses     |
     | sstuser credentials    |
     | stored in wsrep.cnf    |
```

---

**4. Replication account**
```sql
'replication_user'@'%'
```
| Property | Value |
|---|---|
| Purpose | MySQL/MariaDB binary log replication between primary and replica |
| Is a process | ❌ No — account used by the replication thread |
| Used by | The internal replication IO thread |
| Required privileges | `REPLICATION SLAVE`, `REPLICATION CLIENT` |
| Shell access | ❌ No |

---

**5. Backup account**
```sql
'backup_user'@'localhost'
```
| Property | Value |
|---|---|
| Purpose | Used by `mariabackup` or `mysqldump` to take backups |
| Is a process | ❌ No — account used by the backup tool |
| Used by | Backup scripts or cron jobs |
| Required privileges | `RELOAD`, `LOCK TABLES`, `PROCESS`, `REPLICATION CLIENT` |
| Shell access | ❌ No |

---

**6. Monitoring account**
```sql
'zabbix'@'10.19.6.10'
```
| Property | Value |
|---|---|
| Purpose | Used by Zabbix agent to query DB metrics |
| Is a process | ❌ No — account used by Zabbix monitoring process |
| Used by | Zabbix agent running on the DB host |
| Required privileges | `PROCESS`, `REPLICATION CLIENT` — read only |
| Shell access | ❌ No |

---

**7. Application account**
```sql
'app_payments'@'10.19.6.50'
```
| Property | Value |
|---|---|
| Purpose | Used by a specific application to read/write its database |
| Is a process | ❌ No — account used by the application process |
| Used by | Backend application server |
| Required privileges | `SELECT`, `INSERT`, `UPDATE`, `DELETE` on specific DB only |
| Shell access | ❌ No |

---

**8. Human DBA account**
```sql
'dba_alice'@'10.19.6.5'
```
| Property | Value |
|---|---|
| Purpose | Named person performing database administration |
| Is a process | ❌ No — used by a human |
| Required privileges | Administrative — `SUPER`, `RELOAD`, schema changes etc |
| Shell access | ❌ No — DB account only |
| The person also has | A separate OS account for shell access |

---

# Part 3 — How UI and Backend Communicate with the DB

---

**The full communication chain:**

```
User browser
     |
     | HTTPS
     v
Frontend UI (React/Angular etc)
     |
     | HTTP/API calls
     v
Backend application server (Node.js/Python/Java etc)
     |
     | TCP port 3306
     | using DB account credentials
     v
MariaDB server
     |
     | reads/writes
     v
Data files on disk
```

---

**The backend connects as a DB user — never as an OS user:**

```python
# Example Python backend connection
connection = mariadb.connect(
    host="10.19.6.96",
    port=3306,
    user="app_payments",      # This is a DB account
    password="dbpassword",    # DB account password
    database="payments_db"
)
```

| Layer | Account used |
|---|---|
| Browser to frontend | No DB account involved |
| Frontend to backend API | No DB account involved |
| Backend to MariaDB | DB account e.g. `app_payments@10.19.6.96` |
| MariaDB to disk | OS `mysql` service account owns the files |

---

# Part 4 — How Rights are Granted

---

**DB account rights — granted with SQL:**
```sql
-- Create account
CREATE USER 'app_payments'@'10.19.6.50' 
  IDENTIFIED BY 'strongpassword';

-- Grant specific privileges
GRANT SELECT, INSERT, UPDATE, DELETE 
  ON payments_db.* 
  TO 'app_payments'@'10.19.6.50';

-- Apply changes
FLUSH PRIVILEGES;
```

**Privilege levels in MariaDB:**

| Level | Scope | Example |
|---|---|---|
| Global | All databases on server | `GRANT ALL ON *.*` |
| Database | One specific database | `GRANT ALL ON mydb.*` |
| Table | One specific table | `GRANT SELECT ON mydb.users` |
| Column | One specific column | `GRANT SELECT (email) ON mydb.users` |

---

**OS account rights — granted with OS commands:**
```bash
# Add user to a group
usermod -aG mysql leo_audit

# Grant sudo access
visudo
# Add: leo_audit ALL=(ALL) NOPASSWD: /bin/systemctl restart mariadb

# Set file ownership
chown mysql:mysql /var/lib/mysql
chmod 750 /var/lib/mysql
```

---

# Part 5 — How Authentication Works

---

**DB account authentication methods:**

| Plugin | How it works | Strength |
|---|---|---|
| `mysql_native_password` | SHA1 hash of password stored in `mysql.user` | ❌ Weak |
| `ed25519` | Modern elliptic curve — password hashed with ed25519 | ✅ Strong |
| `sha256_password` | SHA256 hash | ✅ Strong |
| `unix_socket` | Authenticates via OS user — no password needed | ✅ Strong for local |
| `gssapi` | Kerberos/Active Directory ticket | ✅ Strong |
| `pam` | Delegates to Linux PAM | ✅ Depends on PAM config |

**unix_socket special case — the one link between OS and DB accounts:**
```sql
-- DB user linked to OS user via unix_socket
CREATE USER 'leo_audit'@'localhost' 
  IDENTIFIED VIA unix_socket;
```
```bash
# OS user leo_audit can now connect without password
mysql -u leo_audit
# MariaDB verifies the connecting OS process is owned by leo_audit
```

---

**OS account authentication methods:**

| Method | How it works |
|---|---|
| Password | Hashed in `/etc/shadow` using SHA512 |
| SSH key | RSA/ED25519 key pair — private key on client, public key on server |
| SSSD + AD | Kerberos ticket from Active Directory domain controller |
| PAM | Chains multiple methods together |

---

# Part 6 — Shell Access Summary

| Account type | Has OS shell | Has DB access | Notes |
|---|---|---|---|
| OS root | ✅ Yes | Only with DB credentials | Most powerful OS account |
| OS `mysql` service account | ❌ No (`/sbin/nologin`) | No direct login | Owns DB files and process |
| OS human account (`leo_audit`) | ✅ Yes | Only with DB credentials | Separate from DB identity |
| OS application account | ❌ Usually no | No direct login | Runs app process |
| DB root | ❌ No | ✅ Full DB access | Separate from OS root |
| DB system reserved accounts | ❌ No | ✅ Internal use only, locked | Cannot authenticate |
| DB `sstuser` | ❌ No | ✅ SST privileges only | Used by Galera process |
| DB backup account | ❌ No | ✅ Backup privileges only | Used by backup tool |
| DB monitoring account | ❌ No | ✅ Read only | Used by Zabbix process |
| DB application account | ❌ No | ✅ App DB only | Used by backend code |
| DB human DBA account | ❌ No | ✅ Administrative | Person also has separate OS account |

---

**The fundamental rule:**

```
OS accounts  → control who can log into the Linux shell
DB accounts  → control who can connect to MariaDB and run SQL
They are completely independent of each other
The only bridge between them is the unix_socket plugin
A person typically needs BOTH:
  - An OS account to SSH into the server
  - A DB account to connect to MariaDB
```
