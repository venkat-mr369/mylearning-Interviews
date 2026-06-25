---

### 1. What is a Checkpoint in MySQL?

### Answer

Checkpoint is the process of flushing dirty pages from Buffer Pool to disk.

Purpose:

* Reduce crash recovery time
* Ensure modified pages are persisted

Flow:

```text
User Update
    ↓
Buffer Pool (Dirty Page)
    ↓
Redo Log
    ↓
Checkpoint
    ↓
Data File (.ibd)
```

### Interview Follow-up

What happens if MySQL crashes before checkpoint?

Answer:

Redo logs are used during crash recovery to replay committed changes.

---

# 2. Difference Between Sharp Checkpoint and Fuzzy Checkpoint

### Sharp Checkpoint

Flush all dirty pages.

Very expensive.

---

### Fuzzy Checkpoint

Flush only some dirty pages.

InnoDB uses fuzzy checkpoints.

Reason:

Avoid performance impact.

---

# 3. What are Dirty Pages?

### Answer

Pages modified in memory but not yet written to disk.

Check:

```sql
SHOW GLOBAL STATUS
LIKE 'Innodb_buffer_pool_pages_dirty';
```

Large dirty pages indicate:

* Heavy write workload
* Checkpoint lag

---

# 4. Explain MySQL Thread Architecture

Every client connection creates a thread.

```text
Client
   ↓
Connection Thread
   ↓
Parser
   ↓
Optimizer
   ↓
Storage Engine
```

Check:

```sql
SHOW PROCESSLIST;
```

---

# 5. What Types of Threads Exist in MySQL?

### Foreground Threads

User Connections

```text
Client 1
Client 2
Client 3
```

---

### Background Threads

Internal InnoDB threads.

Important ones:

```text
Master Thread
IO Thread
Purge Thread
Page Cleaner Thread
Log Thread
Replication Threads
```

---

# 6. What is the Master Thread?

### Answer

Most important InnoDB thread.

Responsibilities:

* Flush dirty pages
* Checkpoint management
* Merge insert buffer
* Purge old records

Think of it as:

```text
Housekeeping Manager
```

---

# 7. What is Page Cleaner Thread?

### Answer

Flushes dirty pages from Buffer Pool.

Before MySQL 5.5:

Master thread handled everything.

Modern versions:

Dedicated Page Cleaner thread.

Check:

```sql
SHOW ENGINE INNODB STATUS\G
```

---

# 8. What is Purge Thread?

### Answer

Removes old MVCC versions.

Example:

Transaction A:

```sql
UPDATE emp
SET salary=10000;
```

Old version cannot be removed immediately.

Purge thread cleans it later.

Monitor:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for:

```text
History List Length
```

---

# 9. What is History List Length?

### Answer

Number of undo records waiting for purge.

Check:

```sql
SHOW ENGINE INNODB STATUS\G
```

Example:

```text
History list length 100000
```

Possible issue:

Long-running transaction.

---

# 10. What is Undo Log?

### Answer

Stores old row versions.

Used for:

* Rollback
* MVCC
* Consistent Reads

Example:

```sql
UPDATE employees
SET salary=10000
WHERE id=1;
```

Old value stored in Undo.

---

# 11. What is Redo Log?

### Answer

Stores changes before writing data pages.

Purpose:

Crash Recovery.

Flow:

```text
Update
   ↓
Redo Log
   ↓
Buffer Pool
   ↓
Checkpoint
```

Files:

```text
ib_logfile0
ib_logfile1
```

or

```text
#innodb_redo
```

(MySQL 8)

---

# 12. Explain WAL (Write Ahead Logging)

### Answer

Redo written before data page.

Rule:

```text
Redo First
Data Later
```

This guarantees durability.

ACID "D".

---

# 13. What Happens Internally During COMMIT?

Example:

```sql
BEGIN;

UPDATE emp
SET salary=10000
WHERE id=1;

COMMIT;
```

Internal Flow:

```text
1. Update Buffer Pool
2. Create Undo
3. Create Redo
4. Write Binlog
5. Flush Redo
6. Commit
7. ACK to User
```

Very common interview question.

---

# 14. Explain Two-Phase Commit (2PC)

Used between:

```text
Redo Log
Binlog
```

Without 2PC:

Replication inconsistency possible.

Flow:

```text
Prepare Redo
      ↓
Write Binlog
      ↓
Commit Redo
```

Guarantees consistency.

---

# 15. What Happens When MySQL Crashes?

Recovery Steps:

```text
1. Read Checkpoint
2. Scan Redo Logs
3. Roll Forward
4. Rollback Uncommitted Transactions
5. Open Database
```

Known as:

```text
Crash Recovery
```

---

# 16. Explain InnoDB IO Threads

Check:

```sql
SHOW VARIABLES
LIKE 'innodb_read_io_threads';
```

and

```sql
SHOW VARIABLES
LIKE 'innodb_write_io_threads';
```

Types:

### Read IO Thread

Reads pages.

### Write IO Thread

Writes pages.

### Log IO Thread

Handles redo writes.

### Insert Buffer Thread

Merge operations.

---

# 17. Explain Change Buffer (Insert Buffer)

Used for:

```text
Secondary Index Changes
```

Instead of immediate disk write:

```text
Store Change
      ↓
Merge Later
```

Benefit:

Less random I/O.

Check:

```sql
SHOW ENGINE INNODB STATUS\G
```

---

# 18. What is Doublewrite Buffer?

### Problem

Partial page writes.

Example:

Power failure during page write.

Page corrupted.

---

### Solution

Doublewrite Buffer.

Flow:

```text
Buffer Pool
      ↓
Doublewrite Buffer
      ↓
Actual Data File
```

Provides recovery protection.

---

# 19. What is Adaptive Hash Index?

### Answer

InnoDB automatically creates hash lookups for hot pages.

Benefit:

Faster reads.

Check:

```sql
SHOW ENGINE INNODB STATUS\G
```

Disable:

```sql
SET GLOBAL
innodb_adaptive_hash_index=OFF;
```

---

# 20. What Threads are Used in Replication?

### Source

Binlog Dump Thread

Check:

```sql
SHOW PROCESSLIST;
```

Output:

```text
Binlog Dump
```

---

### Replica

IO Thread

Reads source binlog.

---

SQL Thread

Executes events.

---

MySQL 8:

```text
Coordinator Thread
Worker Threads
```

Parallel replication.

---

# Important Troubleshooting Questions

### Which thread removes old row versions?

Answer:

```text
Purge Thread
```

---

### Which thread writes dirty pages?

Answer:

```text
Page Cleaner Thread
```

---

### Which thread manages checkpoints?

Answer:

```text
Master Thread
```

---

### Which thread handles replication events?

Answer:

```text
IO Thread
SQL Thread
Coordinator Thread
Worker Threads
```

---

### Which thread writes redo logs?

Answer:

```text
Log Writer Thread
```

---

### Long-running transaction causes which issue?

Answer:

```text
History List Length Growth
Undo Tablespace Growth
Purge Lag
```

---

### Which component helps crash recovery?

Answer:

```text
Redo Log
Checkpoint
Doublewrite Buffer
```

---

### Which component provides rollback?

Answer:

```text
Undo Log
```

---

### Which component provides MVCC?

Answer:

```text
Undo Log + Read View
```

