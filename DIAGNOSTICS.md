# MySQL Production Diagnostics Reference

Queries covering Phase 1 (Storage) and Phase 2 (MVCC, Locking, Transactions).
Organised by the question you are trying to answer, not by topic.

---

## 1. Quick Health Check — Run These First

```sql
-- Overall InnoDB state: buffer pool, redo log, transactions, deadlocks
SHOW ENGINE INNODB STATUS\G
-- Sections to read: BUFFER POOL, LOG, TRANSACTIONS, LATEST DETECTED DEADLOCK
```

```sql
-- What is running right now (non-idle connections)
SELECT id, user, host, db, command, time, state,
       LEFT(info, 120) AS query
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
-- Alarm: any row with time > 30 and state like "Waiting for lock" or "Copying to tmp table"
```

```sql
-- History list length (purge debt — how far the purge thread is behind)
SELECT name, count AS history_list_length
FROM information_schema.innodb_metrics
WHERE name = 'trx_rseg_history_len';
-- Target: < 10,000. Alarm: > 100,000 — find and kill the long-running transaction causing it.
```

---

## 2. Storage Investigation

### Find fragmented tables
```sql
SELECT
  table_schema,
  table_name,
  row_format,
  table_rows,
  ROUND(data_length  / 1024 / 1024, 1) AS data_mb,
  ROUND(data_free    / 1024 / 1024, 1) AS free_mb,
  ROUND(index_length / 1024 / 1024, 1) AS indexes_mb,
  ROUND(data_free / (data_length + data_free) * 100, 1) AS end_fragmentation_pct,
  ROUND(avg_row_length, 0) AS avg_row_bytes
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER BY data_length DESC;
-- end_fragmentation_pct: only shows pre-allocated empty space at end of file.
-- To estimate real data: avg_row_bytes * table_rows ≈ actual row data.
-- Large gap between data_mb and (avg_row_bytes * table_rows / 1024 / 1024) = internal fragmentation.
```

### Find tables with no primary key
```sql
SELECT t.table_schema, t.table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints tc
  ON  t.table_schema = tc.table_schema
  AND t.table_name   = tc.table_name
  AND tc.constraint_type = 'PRIMARY KEY'
WHERE t.table_type = 'BASE TABLE'
  AND tc.constraint_name IS NULL
  AND t.table_schema NOT IN ('mysql','information_schema','performance_schema','sys');
-- Any row here = table using hidden DB_ROW_ID as clustered index.
-- Cost: 19 bytes invisible overhead per row, index you can never query.
```

### Buffer pool hit ratio
```sql
SELECT
  FORMAT((1 - reads / read_requests) * 100, 4) AS hit_ratio_pct,
  reads                                          AS physical_disk_reads,
  read_requests                                  AS total_read_requests
FROM (
  SELECT
    SUM(IF(variable_name = 'Innodb_buffer_pool_reads',
           variable_value, 0)) AS reads,
    SUM(IF(variable_name = 'Innodb_buffer_pool_read_requests',
           variable_value, 0)) AS read_requests
  FROM performance_schema.global_status
  WHERE variable_name IN (
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_read_requests'
  )
) t;
-- Target: > 99%. Below 95% = buffer pool too small for working set.
-- Fix: increase innodb_buffer_pool_size (typically 70-80% of available RAM).
```

### Row format distribution
```sql
SELECT row_format, COUNT(*) AS tables
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
GROUP BY row_format;
-- All tables should be DYNAMIC (default since MySQL 5.7).
-- COMPACT = legacy, less efficient for large columns.
-- REDUNDANT = very old, should not appear in new schemas.
```

---

## 3. MVCC and Transaction Investigation

### Find long-running transactions (the most dangerous thing for MVCC health)
```sql
SELECT
  trx_id,
  trx_state,
  trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
  trx_rows_modified,
  trx_rows_locked,
  trx_isolation_level,
  trx_is_read_only,
  LEFT(trx_query, 120) AS current_query
FROM information_schema.innodb_trx
ORDER BY trx_started ASC;
-- Alarm: any transaction older than 60 seconds.
-- trx_rows_modified=0 + old age = idle read transaction holding a read view.
--   → blocking purge, undo log growing, secondary index reads degrading for all sessions.
-- trx_rows_modified>0 + old age = long-running write transaction.
--   → large undo log, expensive rollback if killed, risk of lock escalation.
```

### Estimate undo log growth from a specific transaction
```sql
-- Run this while a suspicious transaction is open
SELECT
  trx_id,
  trx_started,
  TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS age_sec,
  trx_rows_modified                          AS undo_log_entries
FROM information_schema.innodb_trx
ORDER BY trx_started ASC;
-- undo_log_entries = number of row versions this transaction has written.
-- This is also the cost InnoDB uses for deadlock victim selection:
-- lower undo_log_entries = more likely to be chosen as victim.
```

### Check redo log pressure
```sql
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
  'Innodb_os_log_written',   -- total bytes written to redo log since startup
  'Innodb_log_waits',        -- times a transaction waited for redo log buffer space
  'Innodb_buffer_pool_bytes_dirty'  -- dirty pages not yet flushed to .ibd files
);
-- Innodb_log_waits > 0 = innodb_log_buffer_size is too small for your write rate.
-- Large Innodb_buffer_pool_bytes_dirty = checkpoint pressure,
--   InnoDB is struggling to flush dirty pages fast enough.
```

### Verify durability configuration
```sql
SELECT variable_name, variable_value
FROM performance_schema.global_variables
WHERE variable_name IN (
  'innodb_flush_log_at_trx_commit',  -- 1=safe, 2=OS cache, 0=risky
  'sync_binlog',                     -- 1=safe, 0=OS decides
  'binlog_format',                   -- ROW=safe, STATEMENT=unsafe with RC
  'gtid_mode',                       -- ON = each transaction applied exactly once
  'transaction_isolation'            -- REPEATABLE-READ = default, gap locks active
);
-- Production payment/critical systems: both flush variables must be 1.
-- binlog_format must be ROW for safety with any isolation level.
```

---

## 4. Lock Contention Investigation

### Find who is blocking whom right now
```sql
SELECT
  r.trx_id                                         AS waiting_trx_id,
  LEFT(r.trx_query, 80)                            AS waiting_query,
  TIMESTAMPDIFF(SECOND, r.trx_started, NOW())      AS waiting_for_sec,
  b.trx_id                                         AS blocking_trx_id,
  LEFT(b.trx_query, 80)                            AS blocking_query,
  TIMESTAMPDIFF(SECOND, b.trx_started, NOW())      AS blocker_running_for_sec,
  b.trx_rows_modified                              AS blocker_undo_entries
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b
  ON b.trx_id = w.blocking_engine_transaction_id
JOIN information_schema.innodb_trx r
  ON r.trx_id = w.requesting_engine_transaction_id
ORDER BY waiting_for_sec DESC;
-- If blocker_running_for_sec is large with trx_rows_modified=0 = idle connection
-- holding a lock from a forgotten BEGIN. Kill it.
```

### See all current row locks (with type and what they protect)
```sql
SELECT
  object_schema,
  object_name,
  index_name,
  lock_type,
  lock_mode,
  lock_status,
  lock_data
FROM performance_schema.data_locks
WHERE lock_type = 'RECORD'
ORDER BY object_name, lock_data;
-- lock_mode decoder:
--   X,REC_NOT_GAP = record lock only (unique index exact match)
--   X,GAP         = gap lock (preventing inserts into a range)
--   X             = next-key lock (gap + record, used for range queries)
--   INSERT_INTENTION = waiting to insert into a gap
```

### Find the connection ID holding a lock so you can kill it
```sql
SELECT
  p.id                                        AS connection_id,
  p.user,
  p.host,
  p.time                                      AS connected_sec,
  p.state,
  t.trx_id,
  t.trx_rows_modified,
  LEFT(t.trx_query, 100)                      AS last_query
FROM information_schema.processlist p
JOIN information_schema.innodb_trx t
  ON t.trx_mysql_thread_id = p.id
ORDER BY p.time DESC;

-- To kill a connection:
-- KILL <connection_id>;
-- This rolls back the transaction and releases all its locks.
```

---

## 5. Deadlock Investigation

### Read the last deadlock
```sql
SHOW ENGINE INNODB STATUS\G
-- Find section: LATEST DETECTED DEADLOCK
-- Read structure:
--   (1) TRANSACTION: the first participant
--       HOLDS: what index records it locked
--       WAITING FOR: what it was trying to lock when the cycle was detected
--   (2) TRANSACTION: the second participant (same structure)
--   WE ROLL BACK TRANSACTION (N): which one was killed (lowest undo log entries)
```

### Decode physical record fields in the deadlock output
```
field 0: PRIMARY KEY value         (hex with high bit flipped for signed int)
field 1: db_trx_id  (6 bytes)      which transaction last wrote this row
field 2: db_roll_ptr (7 bytes)     pointer into undo log for previous version
field 3+: your column values
```

```sql
-- Deadlock frequency since last restart
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_deadlocks';
-- > 0 per hour on a production system = investigate application lock ordering.
-- Fix pattern: always acquire locks in the same order across all transactions.
```

---

## Key Numbers to Memorise

| What | Target | Alarm |
|---|---|---|
| Buffer pool hit ratio | > 99% | < 95% |
| History list length | < 10,000 | > 100,000 |
| Longest open transaction | < 10 sec | > 60 sec |
| Innodb_log_waits | 0 | > 0 |
| Innodb_deadlocks / hour | 0 | > 5 |
| Tables with no primary key | 0 | any |
| innodb_flush_log_at_trx_commit (payments) | 1 | anything else |

---

## Hidden Column Sizes (Phase 1 Reference)

```
Table WITH primary key:
  db_trx_id   = 6 bytes   (MVCC: who last wrote this row)
  db_roll_ptr = 7 bytes   (MVCC: pointer to previous version in undo log)
  Total overhead = 13 bytes per row

Table WITHOUT primary key (no PK defined):
  db_row_id   = 6 bytes   (InnoDB-generated hidden clustered index key)
  db_trx_id   = 6 bytes
  db_roll_ptr = 7 bytes
  Total overhead = 19 bytes per row
```
