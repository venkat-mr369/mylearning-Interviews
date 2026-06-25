## MySQL Core DBA Interview Questions & Production Basics (15+ Years DBA Level)

---

## 1. What are the different types of MySQL backups?

### Answer

There are two major types:

### Logical Backup

Uses SQL statements.

```bash
mysqldump -u root -p employees > employees.sql
```

Advantages:

* Human readable
* Easy migration

Disadvantages:

* Slow for large databases

---

### Physical Backup

Copies actual data files.

Examples:

```bash
xtrabackup --backup --target-dir=/backup
```

Advantages:

* Faster
* Suitable for TB-sized databases

Disadvantages:

* Version dependent

---

### Interview Follow-up

**Which backup is preferred for a 2TB production database?**

Answer:
Physical backup using Percona XtraBackup.

---

# 2. Difference Between Cold, Warm and Hot Backup?

### Cold Backup

Database stopped.

```bash
systemctl stop mysqld
```

Copy files.

---

### Warm Backup

Read operations allowed.

Limited writes.

---

### Hot Backup

Database fully online.

Example:

```bash
xtrabackup --backup
```

No downtime.

---

# 3. What is Binary Log?

### Answer

Records all changes made to data.

Used for:

* Replication
* Point-In-Time Recovery (PITR)

Check:

```sql
SHOW VARIABLES LIKE 'log_bin';
```

View logs:

```bash
mysqlbinlog mysql-bin.000001
```

---

# 4. Explain Point-In-Time Recovery (PITR)

### Scenario

Backup taken at 10 AM.

User accidentally deleted data at 2 PM.

### Recovery Steps

Restore backup:

```bash
mysql < backup.sql
```

Apply binlogs until 1:59 PM:

```bash
mysqlbinlog \
--stop-datetime="2026-06-25 13:59:59" \
mysql-bin.000001 | mysql
```

Database restored before deletion.

---

# 5. How do you create a MySQL user?

### Example

```sql
CREATE USER 'appuser'@'%' IDENTIFIED BY 'Password123';
```

Verify:

```sql
SELECT user,host FROM mysql.user;
```

---

# 6. Difference Between '%' and 'localhost'

### localhost

Only local connections.

```sql
CREATE USER 'admin'@'localhost';
```

---

### %

Any host.

```sql
CREATE USER 'admin'@'%';
```

Production recommendation:

Avoid `%` whenever possible.

---

# 7. How do you grant privileges?

### Example

```sql
GRANT SELECT,INSERT,UPDATE
ON sales.*
TO 'appuser'@'%';
```

Apply:

```sql
FLUSH PRIVILEGES;
```

(MySQL 8 usually updates automatically but DBAs still know this command.)

---

# 8. Difference Between REVOKE and DROP USER

### Revoke

Removes permissions.

```sql
REVOKE ALL PRIVILEGES
ON sales.*
FROM 'appuser'@'%';
```

User still exists.

---

### Drop User

Deletes account.

```sql
DROP USER 'appuser'@'%';
```

---

# 9. How do you find all user privileges?

### Query

```sql
SHOW GRANTS FOR 'appuser'@'%';
```

Output:

```sql
GRANT SELECT,INSERT ON sales.*
TO 'appuser'@'%'
```

---

# 10. What is InnoDB?

### Answer

Default MySQL storage engine.

Features:

* Transactions
* Row-level locking
* MVCC
* Crash Recovery
* Foreign Keys

Check:

```sql
SHOW ENGINES;
```

---

# 11. What is the InnoDB Buffer Pool?

### Answer

Caches data pages and indexes in memory.

Check size:

```sql
SHOW VARIABLES
LIKE 'innodb_buffer_pool_size';
```

Production recommendation:

60%-80% of RAM on dedicated DB servers.

Example:

Server RAM = 64GB

Buffer Pool:

```text
48GB
```

---

# 12. How do you identify long-running queries?

### Query

```sql
SHOW FULL PROCESSLIST;
```

or

```sql
SELECT *
FROM information_schema.processlist
ORDER BY time DESC;
```

Kill query:

```sql
KILL 12345;
```

---

# 13. What is a Deadlock?

### Example

Transaction 1

```sql
UPDATE accounts
SET balance=100
WHERE id=1;
```

Transaction 2

```sql
UPDATE accounts
SET balance=200
WHERE id=2;
```

Then both try updating the other's row.

MySQL detects deadlock and rolls back one transaction.

Check:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look under:

```text
LATEST DETECTED DEADLOCK
```

---

# 14. What is a Tablespace?

### Answer

Logical storage structure holding table data.

Check:

```sql
SELECT *
FROM information_schema.files;
```

Individual tables:

```sql
innodb_file_per_table=ON
```

Creates:

```text
table.ibd
```

per table.

---

# 15. What happens when a row is deleted?

### Example

```sql
DELETE FROM employees
WHERE emp_id=100;
```

Internally:

1. Row marked deleted
2. Undo generated
3. Redo generated
4. Binary log written
5. Commit

Space is NOT immediately released.

Reclaim:

```sql
OPTIMIZE TABLE employees;
```

---

# 16. Difference Between DELETE, TRUNCATE and DROP

### DELETE

```sql
DELETE FROM emp;
```

* DML
* Can rollback

---

### TRUNCATE

```sql
TRUNCATE TABLE emp;
```

* Fast
* Resets AUTO_INCREMENT

---

### DROP

```sql
DROP TABLE emp;
```

Removes table completely.

---

# 17. How do you check database size?

### Database Size

```sql
SELECT table_schema,
ROUND(SUM(data_length+index_length)/1024/1024,2) MB
FROM information_schema.tables
GROUP BY table_schema;
```

---

### Table Size

```sql
SELECT table_name,
ROUND((data_length+index_length)/1024/1024,2) MB
FROM information_schema.tables
WHERE table_schema='sales';
```

---

# 18. How do you monitor replication?

### MySQL 5.7

```sql
SHOW SLAVE STATUS\G
```

Important fields:

```text
Slave_IO_Running
Slave_SQL_Running
Seconds_Behind_Master
```

---

### MySQL 8.0

```sql
SHOW REPLICA STATUS\G
```

Healthy:

```text
Replica_IO_Running: Yes
Replica_SQL_Running: Yes
```

---

# 19. What is Seconds_Behind_Master?

### Answer

Replication lag.

Check:

```sql
SHOW REPLICA STATUS\G
```

Example:

```text
Seconds_Behind_Master: 300
```

Meaning:

Replica is 5 minutes behind.

Possible causes:

* Long query
* Large transaction
* Network issue
* Slow disk

---

# 20. Production Issue: Disk Full

### Symptoms

```text
ERROR 1114:
The table is full
```

Check:

```bash
df -h
```

Check large files:

```bash
du -sh /var/lib/mysql/*
```

Check binlogs:

```sql
SHOW BINARY LOGS;
```

Purge old logs:

```sql
PURGE BINARY LOGS
TO 'mysql-bin.000150';
```

---

# Production DBA Daily Health Checks

### Server Uptime

```sql
SHOW GLOBAL STATUS
LIKE 'Uptime';
```

### Connections

```sql
SHOW GLOBAL STATUS
LIKE 'Threads_connected';
```

### Running Queries

```sql
SHOW FULL PROCESSLIST;
```

### Replication

```sql
SHOW REPLICA STATUS\G
```

### Database Size

```sql
SELECT table_schema,
ROUND(SUM(data_length+index_length)/1024/1024/1024,2) GB
FROM information_schema.tables
GROUP BY table_schema;
```

### Error Log Review

```bash
tail -100 /var/log/mysqld.log
```

---

These 20 questions cover roughly **70-80% of what a production MySQL Core DBA is asked in interviews**, especially around **backups, recovery, users, privileges, storage, InnoDB, replication, monitoring, and troubleshooting**. The next level would be deeper topics like **MVCC, Undo Logs, Redo Logs, Buffer Pool Internals, Change Buffer, Doublewrite Buffer, Purge Threads, Metadata Locks (MDL), Online DDL, GTID Replication, and InnoDB Crash Recovery**.
