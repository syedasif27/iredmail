
```markdown
# MySQL Replication Setup for iRedMail Servers with Load Balanced Web Layer

This guide explains how to configure MySQL/MariaDB replication between two iRedMail servers and use a load balancer for high availability webmail access.

---

## 🧱 Prerequisites

- Two iRedMail servers with the same MySQL/MariaDB version.
- Static IP addresses or DNS names for both servers.
- Port 3306 open between both servers.
- MySQL root access on both servers.

---

## 🔧 Step 1: Configure the Master (Primary iRedMail Server)

### 1.1 Edit MySQL configuration

Edit `/etc/mysql/my.cnf` or `/etc/my.cnf`:

```ini
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log

# Replicate specific databases
binlog_do_db = vmail
binlog_do_db = iredadmin
binlog_do_db = roundcubemail  # Optional, if using Roundcube
```

Restart MySQL:
```bash
sudo systemctl restart mysql
```

### 1.2 Create a Replication User

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY 'StrongPassword';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
```

### 1.3 Lock Tables and Get Master Status

```sql
FLUSH TABLES WITH READ LOCK;
SHOW MASTER STATUS;
```

Note the output:
- `File`: e.g. `mysql-bin.000001`
- `Position`: e.g. `154`

Keep the session open to maintain the lock.

### 1.4 Dump the Databases

```bash
mysqldump -u root -p --databases vmail iredadmin roundcubemail > /root/iredmail.sql
```

Then unlock tables:

```sql
UNLOCK TABLES;
```

---

## 🔧 Step 2: Configure the Slave (New iRedMail Server)

### 2.1 Copy and Import the Database Dump

```bash
scp root@master-server:/root/iredmail.sql /root/
mysql -u root -p < /root/iredmail.sql
```

### 2.2 Edit Slave MySQL Configuration

```ini
[mysqld]
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin.log
```

Restart MySQL:
```bash
sudo systemctl restart mysql
```

### 2.3 Set Up Replication

```sql
CHANGE MASTER TO
  MASTER_HOST='MASTER_IP',
  MASTER_USER='repl',
  MASTER_PASSWORD='StrongPassword',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;

START SLAVE;
```

### ✅ Verify Replication

```sql
SHOW SLAVE STATUS\\G
```

Check that:
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`

---

## 🔁 Load Balancer for Web Layer (Roundcube, iRedAdmin, etc.)

Use HAProxy or Nginx to distribute traffic across both servers.

### Example HAProxy Config

```haproxy
frontend webmail
    bind *:80
    bind *:443 ssl crt /etc/ssl/private/your.pem
    mode http
    default_backend roundcube_backends

backend roundcube_backends
    mode http
    balance roundrobin
    option httpchk
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
```

---

## 🔐 Security Notes

- Restrict `'repl'@'%'` to only the slave IP.
- Use SSL for MySQL replication in production.

---

## 📦 Related iRedMail Directories

- `/var/vmail/`: Maildir (should be shared or replicated using Dovecot or FS)
- `/etc/postfix/`, `/etc/dovecot/`: Config files (manage via Ansible or manually)

---
