# MySQL Deep Learning Roadmap

**Lab**: MySQL 8.4 — Primary (`:33050`) + Replica (`:33051`), GTID replication, Performance Schema, Prometheus + Grafana

Each phase builds on the previous. The lab exercises assume your Docker stack is running.

---

## Phase 1 — Storage Internals: How InnoDB Stores Data

**Goal**: Understand how rows actually live on disk before you touch any SQL.

### Topics
- InnoDB page structure (16 KB pages, page header, infimum/supremum, records, free space)
- Row formats: `COMPACT`, `DYNAMIC`, `COMPRESSED` — when each is used, overflow pages
- Tablespace files: `ibdata1`, `.ibd` per-table, undo tablespaces, redo log files
- The double-write buffer — why it exists (partial page writes)
- `innodb_page_size` and its performance implications

### Key Questions to Answer
1. What is stored in `ibdata1` vs `table.ibd`?
2. How does InnoDB handle a row larger than a page (VARCHAR/TEXT/BLOB)?
3. What is the fill factor of a B+Tree leaf page and why does it matter for inserts?

### Lab Exercises
```sql
-- Connect to primary
-- mysql -h 127.0.0.1 -P 33050 -uroot -proot

CREATE DATABASE lab;
USE lab;

-- Observe row format
CREATE TABLE t1 (id INT PRIMARY KEY, name VARCHAR(200)) ROW_FORMAT=DYNAMIC;
SHOW CREATE TABLE t1\G

-- Check space usage per table
SELECT table_name, data_length, index_length, row_format
FROM information_schema.tables
WHERE table_schema = 'lab';

-- See InnoDB status (buffer pool, redo log, etc)
SHOW ENGINE INNODB STATUS\G
```

### Reference
- MySQL 8 Docs: *InnoDB On-Disk Structures*
- `innodb_ruby` tool — read `.ibd` files directly at byte level

---

## Phase 2 — Indexes: Structure, Types, and Internals

**Goal**: Know exactly what an index is on disk, not just "it makes queries faster."

### Topics
- B+Tree structure — internal nodes (key pointers) vs leaf nodes (data)
- **Clustered index** (primary key) — the table IS the B+Tree; row data is in leaf nodes
- **Secondary indexes** — leaf nodes hold the primary key, not the row; double-lookup cost
- **Covering index** — when a secondary index leaf has all needed columns (no row lookup)
- **Composite indexes** — left-prefix rule, column order, selectivity placement
- **Prefix indexes** — on CHAR/VARCHAR/BLOB, trade-off vs covering
- **Functional indexes** — index on expression e.g. `(YEAR(created_at))`
- **Invisible indexes** — test dropping an index without dropping it
- Index selectivity and cardinality — `SHOW INDEX FROM t`
- Index dive vs statistics for range scans (`eq_range_index_dive_limit`)

### Key Questions to Answer
1. If your primary key is a UUID, why can page splits happen more often than with auto-increment?
2. For query `WHERE a=1 AND b>5 ORDER BY b`, what index is optimal?
3. Why does `SELECT *` prevent index-only scans?

### Lab Exercises
```sql
CREATE TABLE orders (
  id        BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id   INT NOT NULL,
  status    TINYINT NOT NULL,
  amount    DECIMAL(10,2),
  created_at DATETIME NOT NULL
);

-- Seed data
INSERT INTO orders (user_id, status, amount, created_at)
  SELECT
    FLOOR(RAND()*10000),
    FLOOR(RAND()*5),
    RAND()*1000,
    NOW() - INTERVAL FLOOR(RAND()*365) DAY
  FROM information_schema.columns c1
  CROSS JOIN information_schema.columns c2
  LIMIT 500000;

-- Test different index strategies
ALTER TABLE orders ADD INDEX idx_user (user_id);
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
ALTER TABLE orders ADD INDEX idx_covering (user_id, status, amount, created_at);

EXPLAIN SELECT amount FROM orders WHERE user_id = 42 AND status = 1;
-- Look at: key, key_len, Extra (Using index = covering)

-- Invisible index test
ALTER TABLE orders ALTER INDEX idx_user INVISIBLE;
EXPLAIN SELECT amount FROM orders WHERE user_id = 42;
ALTER TABLE orders ALTER INDEX idx_user VISIBLE;
```

### Metrics to Watch
```sql
-- Index usage stats (performance_schema)
SELECT object_schema, object_name, index_name, count_read, count_fetch
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'lab'
ORDER BY count_read DESC;
```

---

## Phase 3 — EXPLAIN and the Query Optimizer

**Goal**: Read any query plan fluently and understand why the optimizer made its choices.

### Topics
- `EXPLAIN` output columns: `type`, `key`, `rows`, `filtered`, `Extra`
- Access types ranked: `system` → `const` → `eq_ref` → `ref` → `range` → `index` → `ALL`
- `EXPLAIN FORMAT=JSON` — cost estimates per operation
- `EXPLAIN FORMAT=TREE` (MySQL 8.0.16+) — iterator model
- `EXPLAIN ANALYZE` — actual vs estimated rows (MySQL 8.0.18+)
- Cost model: `engine_cost`, `server_cost` tables in `mysql` schema
- Table statistics: `ANALYZE TABLE`, histograms (`ANALYZE TABLE ... UPDATE HISTOGRAM`)
- Join algorithms: Nested Loop, Hash Join (MySQL 8.0.18+), BKA
- Subquery strategies: materialization vs dependent subquery vs semi-join
- Optimizer hints: `/*+ NO_INDEX_MERGE() */`, `/*+ JOIN_ORDER() */`, `/*+ HASH_JOIN() */`
- `optimizer_switch` flags

### Key Questions to Answer
1. The optimizer estimated 100 rows but scanned 500k — what went wrong and how do you fix it?
2. When does MySQL choose a full index scan over a range scan?
3. What is a "dependent subquery" (DEPENDENT SUBQUERY in EXPLAIN) and why is it slow?

### Lab Exercises
```sql
-- EXPLAIN ANALYZE — shows actual rows vs estimated
EXPLAIN ANALYZE
SELECT u.user_id, COUNT(*), SUM(amount)
FROM orders u
WHERE status IN (1, 2)
  AND created_at > NOW() - INTERVAL 30 DAY
GROUP BY u.user_id
ORDER BY SUM(amount) DESC
LIMIT 10;

-- Build a histogram for low-cardinality column
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 5 BUCKETS;

-- View histogram
SELECT * FROM information_schema.column_statistics
WHERE schema_name='lab' AND table_name='orders'\G

-- Compare optimizer cost model
SELECT * FROM mysql.engine_cost;
SELECT * FROM mysql.server_cost;

-- Force hash join and compare
EXPLAIN FORMAT=TREE
SELECT /*+ HASH_JOIN(o) */ o.user_id, COUNT(*)
FROM orders o
JOIN orders o2 ON o.user_id = o2.user_id
WHERE o.status = 1
LIMIT 100;
```

---

## Phase 4 — Transactions, MVCC, and Locking

**Goal**: Understand how MySQL handles concurrent writes without data corruption.

### Topics
- ACID in InnoDB — which parts are "free" and which have cost
- Isolation levels: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ` (default), `SERIALIZABLE`
- **MVCC** (Multi-Version Concurrency Control): undo log versions, read views, visible-to logic
- Undo log: what it stores, rollback segments, undo tablespace purge
- Redo log (WAL): write-ahead, `innodb_flush_log_at_trx_commit` values 0/1/2
- Lock types: shared (S), exclusive (X), intention locks (IS, IX)
- **Row lock** vs **gap lock** vs **next-key lock** — why gap locks exist
- `SELECT ... FOR UPDATE` vs `SELECT ... FOR SHARE`
- Deadlock detection and the deadlock graph
- Lock wait timeout vs deadlock

### Key Questions to Answer
1. Two transactions in REPEATABLE READ both read the same row, then both update it — what happens?
2. Why can phantom reads occur in READ COMMITTED but not REPEATABLE READ?
3. When does a gap lock get acquired? How do you disable it (and why is that risky)?

### Lab Exercises
```sql
-- Session 1
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- Do NOT commit yet

-- Session 2 (new connection)
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- Should block, then timeout or deadlock

-- Check who holds what lock
SELECT r.trx_id waiting_trx_id, r.trx_query waiting_query,
       b.trx_id blocking_trx_id, b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;

-- View lock details (performance_schema)
SELECT * FROM performance_schema.data_locks\G
SELECT * FROM performance_schema.data_lock_waits\G

-- Simulate deadlock
-- Session 1: BEGIN; UPDATE orders SET status=1 WHERE id=1;
-- Session 2: BEGIN; UPDATE orders SET status=2 WHERE id=2;
-- Session 1: UPDATE orders SET status=1 WHERE id=2;   -- waits
-- Session 2: UPDATE orders SET status=2 WHERE id=1;   -- deadlock

SHOW ENGINE INNODB STATUS\G -- read TRANSACTION section, LATEST DETECTED DEADLOCK
```

---

## Phase 5 — Replication Internals (Your Lab is Already Set Up)

**Goal**: Understand exactly what happens between primary and replica, byte by byte.

### Topics
- Binary log: format (`ROW` vs `STATEMENT` vs `MIXED`), log events, binlog file + position
- GTID (Global Transaction ID): `source_id:sequence_number`, `gtid_executed`, `gtid_purged`
- Replication threads: binlog dump (primary), IO thread (replica), SQL thread (replica)
- Parallel replication: `replica_parallel_workers`, `replica_parallel_type` (`LOGICAL_CLOCK`)
- Replication lag: causes (large transactions, single-threaded apply, DDL), measurement
- Semi-synchronous replication: `rpl_semi_sync_*`
- Delayed replication: `CHANGE REPLICATION SOURCE TO SOURCE_DELAY=N`
- Common replication errors and resolution
- GTID failover vs position-based failover

### Lab Exercises
```bash
# Primary
mysql -h 127.0.0.1 -P 33050 -uroot -proot
```
```sql
-- Check primary status
SHOW BINARY LOG STATUS\G          -- current binlog file + position
SHOW BINARY LOGS;                 -- all binlog files
SHOW GTID_EXECUTED;               -- all committed GTIDs

-- Read raw binlog events
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;

-- On replica
SHOW REPLICA STATUS\G
-- Key fields: Replica_IO_Running, Replica_SQL_Running,
--             Seconds_Behind_Source, Executed_Gtid_Set

-- Test delayed replica
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY=60;
START REPLICA;
SHOW REPLICA STATUS\G  -- SQL_Delay=60

-- Parallel replication config (on replica)
STOP REPLICA SQL_THREAD;
SET GLOBAL replica_parallel_workers = 4;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
START REPLICA SQL_THREAD;

-- Measure replication lag via performance_schema
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
```

---

## Phase 6 — Performance Schema and Monitoring

**Goal**: Pinpoint performance bottlenecks using built-in instrumentation and your Grafana dashboards.

### Topics
- Performance Schema architecture: instruments, consumers, setup tables
- Key tables: `events_statements_summary_by_digest`, `table_io_waits_summary_by_index_usage`
- `sys` schema — human-readable views built on Performance Schema
- Slow query log: `slow_query_log`, `long_query_time`, `log_queries_not_using_indexes`
- `pt-query-digest` (Percona Toolkit) — aggregate slow log
- InnoDB metrics: `SHOW ENGINE INNODB STATUS` sections
- Buffer pool hit ratio — target > 99%
- Prometheus metrics from `mysqld-exporter` (already running on `:9104`, `:9105`)
- Grafana dashboards: import MySQL 8 dashboard (ID `7362`)

### Key Metrics to Track
| Metric | Good | Warning |
|---|---|---|
| Buffer pool hit ratio | > 99% | < 95% |
| Slow queries / sec | 0 | > 1 |
| Threads connected | < 80% of max | > 90% |
| Replication lag (seconds_behind_source) | 0 | > 5 |
| Deadlocks / hour | 0 | > 5 |

### Lab Exercises
```sql
-- Top 10 slowest query digests
SELECT digest_text, count_star, avg_timer_wait/1e12 avg_sec,
       max_timer_wait/1e12 max_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY avg_timer_wait DESC
LIMIT 10\G

-- Worst table I/O
SELECT * FROM sys.io_global_by_wait_by_bytes LIMIT 10;

-- Index usage — find unused indexes
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'lab';

-- Buffer pool hit ratio
SELECT
  (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio_pct
FROM (
  SELECT
    VARIABLE_VALUE AS Innodb_buffer_pool_reads
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) r,
(
  SELECT
    VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) rr;

-- Enable per-thread statement history
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE 'events_statements%';
```

---

## Phase 7 — Schema Design and Advanced SQL

**Goal**: Understand how schema decisions compound into performance wins or disasters at scale.

### Topics
- Choosing the right primary key: auto-increment BIGINT vs UUID vs UUID_v7
- Normalization vs denormalization trade-offs at scale
- Partitioning: `RANGE`, `LIST`, `HASH`, `KEY` — partition pruning, partition-wise operations
- Generated/virtual columns + indexing expressions
- JSON column type vs normalized tables — when to use each
- Full-text search: `FULLTEXT` indexes, `MATCH ... AGAINST`, boolean mode
- Common Table Expressions (CTEs), recursive CTEs
- Window functions: `ROW_NUMBER`, `LAG/LEAD`, `NTILE`, running totals
- Temporal tables and `AS OF` queries (point-in-time reads)

### Lab Exercises
```sql
-- UUID vs auto-increment B+Tree fragmentation
CREATE TABLE t_uuid (id CHAR(36) DEFAULT (UUID()) PRIMARY KEY, val INT);
CREATE TABLE t_auto (id BIGINT AUTO_INCREMENT PRIMARY KEY, val INT);

-- Insert 100k rows to each, compare fragmentation
INSERT INTO t_uuid (val) SELECT RAND()*1000 FROM orders LIMIT 100000;
INSERT INTO t_auto (val) SELECT RAND()*1000 FROM orders LIMIT 100000;

SELECT table_name, data_length, data_free
FROM information_schema.tables
WHERE table_schema='lab' AND table_name IN ('t_uuid','t_auto');

-- Partitioning with pruning
CREATE TABLE events (
  id BIGINT AUTO_INCREMENT,
  created_at DATE NOT NULL,
  payload JSON,
  PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2022 VALUES LESS THAN (2023),
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);

-- Verify partition pruning
EXPLAIN SELECT * FROM events WHERE created_at BETWEEN '2023-01-01' AND '2023-12-31';
-- partitions column should show only p2023
```

---

## Phase 8 — Configuration Tuning and Production Hardening

**Goal**: Know which knobs to turn, why, and how to validate the effect.

### Key Configuration Groups

**Memory**
```ini
innodb_buffer_pool_size         # 70-80% of RAM for dedicated servers
innodb_buffer_pool_instances    # 1 per GB of buffer pool, max 16
innodb_log_buffer_size          # 64M-256M for write-heavy workloads
```

**Durability vs Performance Trade-off**
```ini
innodb_flush_log_at_trx_commit  # 1=safe, 2=faster (OS crash risk), 0=fastest (crash risk)
sync_binlog                     # 1=safe, 0=faster
innodb_doublewrite              # ON=safe, OFF for ephemeral/test
```

**Replication**
```ini
replica_parallel_workers        # 4-16 for parallel apply
replica_parallel_type           # LOGICAL_CLOCK
binlog_expire_logs_seconds      # 604800 (7 days)
```

**Connections**
```ini
max_connections                 # size for your connection pool + headroom
thread_stack                    # 256K (reduce if max_connections is very high)
```

### Lab Exercises
```sql
-- Find all non-default variables (your current tuning)
SELECT variable_name, variable_value
FROM performance_schema.global_variables
WHERE variable_value != (
  SELECT variable_value FROM performance_schema.variables_info
  WHERE variable_name = performance_schema.global_variables.variable_name
  AND variable_source = 'COMPILED'
)
LIMIT 50;

-- Watch InnoDB checkpoint age (redo log pressure)
SELECT
  variable_value / (1024*1024) AS value_mb,
  variable_name
FROM performance_schema.global_status
WHERE variable_name IN (
  'Innodb_redo_log_capacity_resized',
  'Innodb_buffer_pool_bytes_dirty',
  'Innodb_buffer_pool_bytes_data'
);
```

---

## Suggested Study Order

```
Week 1-2   →  Phase 1 (Storage internals)
Week 3-4   →  Phase 2 (Indexes)
Week 5-6   →  Phase 3 (Optimizer + EXPLAIN)
Week 7     →  Phase 4 (MVCC + locking)
Week 8     →  Phase 5 (Replication)
Week 9     →  Phase 6 (Monitoring — wire up Grafana dashboard ID 7362)
Week 10    →  Phase 7 (Schema design)
Week 11-12 →  Phase 8 (Tuning) + revisit earlier phases with new depth
```

---

## Supplemental Resources

| Resource | What It Teaches |
|---|---|
| *MySQL Internals Manual* (dev.mysql.com) | Source-level internals |
| *High Performance MySQL, 4th ed.* (Schwartz, Tkachenko) | Battle-tested tuning wisdom |
| Percona blog (percona.com/blog) | Real-world war stories |
| `EXPLAIN FORMAT=TREE` deep dives on planetmysql.org | Optimizer internals |
| Percona Toolkit: `pt-query-digest`, `pt-online-schema-change` | Production tooling |
| `innodb_ruby` gem | Read `.ibd` files at byte level |

---

## Quick Reference: Your Lab Endpoints

| Service | Port | Credentials |
|---|---|---|
| MySQL Primary | 33050 | root / root |
| MySQL Replica | 33051 | root / root |
| Prometheus | 9090 | — |
| Grafana | 3000 | admin / admin |
| mysqld-exporter (primary) | 9104 | — |
| mysqld-exporter (replica) | 9105 | — |
