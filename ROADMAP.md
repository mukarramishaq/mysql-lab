# MySQL Deep Learning Roadmap

**Lab**: MySQL 8.4 — Primary (`:33050`) + Replica (`:33051`), GTID replication, Performance Schema, Prometheus + Grafana

Each phase builds directly on the previous one. Do not skip phases — the mental models stack.

---

## Lab Bootstrap

Run this once before starting Phase 1. Verify your stack is up and create the learning database.

```bash
# Start the lab
cd ~/Documents/mysql-lab
docker compose up -d

# Verify both instances are running
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

# Connect helpers (save these as aliases)
alias mysql-primary='mysql -h 127.0.0.1 -P 33050 -uroot -proot'
alias mysql-replica='mysql -h 127.0.0.1 -P 33051 -uroot -proot'
```

```sql
-- On primary: create the lab schema used across all phases
CREATE DATABASE IF NOT EXISTS lab;
USE lab;
```

---

## Phase 1 — InnoDB Storage: How Data Lives on Disk

**Goal**: See the physical reality of what MySQL writes to disk before touching any query tuning.

**Why first**: Every concept that follows — indexes, MVCC, locking, replication — is a layer on top of this physical foundation. Understanding pages and tablespaces means EXPLAIN output, buffer pool hit ratios, and redo log pressure will make sense immediately instead of being abstract metrics.

### Topics
- InnoDB page anatomy: 16 KB fixed size, page header, infimum/supremum pseudo-records, user records, free space, page trailer
- Row formats: `COMPACT`, `DYNAMIC` (default), `COMPRESSED` — when each is used, how off-page storage works for large columns
- Tablespace layout: `ibdata1` (system tablespace), per-table `.ibd` files, undo tablespaces (`undo_001`, `undo_002`), redo log (`#ib_redo*`)
- The double-write buffer: why it exists (torn page writes), where it lives in MySQL 8.0+
- B+Tree fill factor: pages fill to ~15/16 on sequential insert, ~1/2 on random insert — why this matters for writes

### Key Questions to Answer
1. What exactly is stored in `ibdata1` vs `lab/orders.ibd`?
2. When you insert a VARCHAR(5000) value, where does the data actually go?
3. Why does inserting with a UUID primary key cause more I/O than an auto-increment primary key?

### Lab Exercises

**Exercise 1.1 — Observe file layout on disk**
```bash
# Look at the actual files MySQL created
docker exec mysql-primary ls -lh /var/lib/mysql/
docker exec mysql-primary ls -lh /var/lib/mysql/#innodb_redo/
# Note: undo_001, undo_002 are undo tablespaces
# #ib_redo files are the redo log (WAL)
```

**Exercise 1.2 — Create tables and watch the .ibd files appear**
```sql
USE lab;
CREATE TABLE t_compact  (id INT PRIMARY KEY, name VARCHAR(200)) ROW_FORMAT=COMPACT;
CREATE TABLE t_dynamic  (id INT PRIMARY KEY, name VARCHAR(200)) ROW_FORMAT=DYNAMIC;
CREATE TABLE t_overflow (id INT PRIMARY KEY, big_col TEXT)      ROW_FORMAT=DYNAMIC;
```
```bash
docker exec mysql-primary ls -lh /var/lib/mysql/lab/
# Each table has its own .ibd file
```

**Exercise 1.3 — Use ibd2sdi to read raw tablespace metadata**
```bash
# ibd2sdi reads the Serialized Dictionary Information from .ibd files
docker exec mysql-primary ibd2sdi /var/lib/mysql/lab/t_dynamic.ibd
# Look for "columns", "indexes", "row_format" in the JSON output

# Insert data and watch the file grow
```
```sql
INSERT INTO t_dynamic (id, name) VALUES (1, 'hello'), (2, 'world');
```
```bash
docker exec mysql-primary ls -lh /var/lib/mysql/lab/t_dynamic.ibd
# Still 96KB — InnoDB pre-allocates extents (64 pages = 1MB minimum)
```

**Exercise 1.4 — Observe overflow pages (off-page storage)**
```sql
-- A DYNAMIC row stores the first 768 bytes inline, rest goes off-page
INSERT INTO t_overflow (id, big_col) VALUES (1, REPEAT('x', 100));
INSERT INTO t_overflow (id, big_col) VALUES (2, REPEAT('x', 10000));
```
```bash
docker exec mysql-primary ls -lh /var/lib/mysql/lab/t_overflow.ibd
# The 10000-byte row causes off-page storage — file will be larger than expected
```

**Exercise 1.5 — Read the InnoDB engine status and understand its sections**
```sql
SHOW ENGINE INNODB STATUS\G
-- Sections to understand now: BUFFER POOL, LOG, TRANSACTIONS
-- You'll revisit this in every subsequent phase
```

**Exercise 1.6 — Observe page fill behavior (sequential vs random)**
```sql
-- Sequential inserts (auto-increment) — pages fill near-completely
CREATE TABLE seq_test (id BIGINT AUTO_INCREMENT PRIMARY KEY, pad CHAR(100));
INSERT INTO seq_test (pad) SELECT REPEAT('a',100) FROM information_schema.columns LIMIT 5000;

SELECT data_length, data_free,
       ROUND(data_free / data_length * 100, 1) AS fragmentation_pct
FROM information_schema.tables
WHERE table_schema = 'lab' AND table_name = 'seq_test';

-- Random inserts (UUID) — pages fill to ~50%, causing fragmentation
CREATE TABLE rnd_test (id CHAR(36) DEFAULT (UUID()) PRIMARY KEY, pad CHAR(100));
INSERT INTO rnd_test (pad) SELECT REPEAT('a',100) FROM information_schema.columns LIMIT 5000;

SELECT data_length, data_free,
       ROUND(data_free / data_length * 100, 1) AS fragmentation_pct
FROM information_schema.tables
WHERE table_schema = 'lab' AND table_name = 'rnd_test';
-- Compare fragmentation_pct between the two tables
```

### Checkpoint
You should now be able to answer: when a user runs `SELECT * FROM orders WHERE id=42`, what physical path does InnoDB take from the `.ibd` file to return the row? (Answer: open the tablespace file → navigate the B+Tree pages to find id=42 → return the row from the leaf page. You'll deepen this in Phase 3.)

---

## Phase 2 — Transactions, MVCC, and Locking

**Goal**: Understand how InnoDB handles concurrent access without corrupting data and without blocking reads.

**Why second**: MVCC is built into the storage layer — the clustered index leaf node doesn't just hold a row, it holds the current version of a row plus a pointer to older versions in the undo log. You need to know this before studying indexes (Phase 3), otherwise you'll learn a simplified picture of what's stored in a B+Tree leaf that you'll have to unlearn later.

### Topics
- **ACID** in InnoDB: which guarantees are "free" (isolation via MVCC) and which cost I/O (durability via redo log)
- **Isolation levels**: `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ` (default), `SERIALIZABLE` — what each reads and when
- **MVCC**: read views, undo log version chains, how MySQL decides which row version is visible to a given transaction
- **Undo log**: what it stores, rollback segments, purge thread (why long-running transactions are dangerous)
- **Redo log (WAL)**: write-ahead logging, `innodb_flush_log_at_trx_commit` values 0/1/2 and their crash-safety trade-offs
- **Lock types**: shared (S), exclusive (X), intention locks (IS, IX) on tables
- **Row-level locks**: record lock, gap lock, next-key lock — why gap locks exist (phantom read prevention)
- `SELECT ... FOR UPDATE` vs `SELECT ... FOR SHARE` — use cases and pitfalls
- **Deadlock** detection, the deadlock graph, `innodb_deadlock_detect`
- Lock wait timeout vs deadlock — how to tell them apart

### Key Questions to Answer
1. Two transactions in REPEATABLE READ both do `SELECT amount FROM orders WHERE id=1` and get 100. Then both do `UPDATE orders SET amount=amount+10 WHERE id=1`. What is the final value and why?
2. Why can phantom reads occur in READ COMMITTED but not REPEATABLE READ?
3. Why is a long-running idle transaction dangerous even if it's doing nothing?

### Lab Exercises

**Exercise 2.1 — Observe MVCC: same row, different visible versions**

Open two terminals.
```sql
-- Terminal 1
BEGIN;
SELECT amount FROM orders WHERE id = 1;  -- you'll seed orders in Phase 3; use t_dynamic for now
-- Note the value, do NOT commit yet

-- Terminal 2
BEGIN;
UPDATE t_dynamic SET name = 'changed' WHERE id = 1;
COMMIT;

-- Back in Terminal 1 (still in same transaction)
SELECT name FROM t_dynamic WHERE id = 1;
-- REPEATABLE READ: you still see the OLD value — MVCC is serving it from the undo log
COMMIT;

-- Now in Terminal 1 again (new transaction)
SELECT name FROM t_dynamic WHERE id = 1;
-- Now you see 'changed' — new read view created at BEGIN
```

**Exercise 2.2 — Observe the undo log growing under a long transaction**
```sql
-- Terminal 1: start a long transaction and do lots of updates
BEGIN;
UPDATE t_dynamic SET name = REPEAT('b', 100) WHERE id IN (1, 2);

-- Terminal 2: watch undo log size
SELECT name, subsystem, count, comment
FROM information_schema.innodb_metrics
WHERE name LIKE 'trx_undo%' OR name LIKE 'purge%';

-- Also check history list length (the "debt" the purge thread must clean up)
SHOW ENGINE INNODB STATUS\G
-- Find: "History list length N" in the TRANSACTIONS section
-- This number grows as long transactions hold snapshots
```

**Exercise 2.3 — Row locks, gap locks, and next-key locks**
```sql
-- Create a simple table with gaps in the primary key
CREATE TABLE lock_demo (id INT PRIMARY KEY, val INT);
INSERT INTO lock_demo VALUES (1,10),(5,50),(10,100),(20,200);

-- Terminal 1
BEGIN;
SELECT * FROM lock_demo WHERE id = 5 FOR UPDATE;
-- This acquires a next-key lock on (1, 5] — meaning gap (1,5) + record 5

-- Terminal 2
BEGIN;
INSERT INTO lock_demo VALUES (3, 30);
-- BLOCKS — id=3 falls in the gap (1,5) which Terminal 1 locked

-- Observe the lock
SELECT engine_lock_id, lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks;
-- lock_mode: X,REC_NOT_GAP = record lock only
-- lock_mode: X,GAP = gap lock only
-- lock_mode: X = next-key lock (gap + record)

-- Clean up
ROLLBACK; -- in both terminals
```

**Exercise 2.4 — Create, observe, and diagnose a deadlock**
```sql
-- Terminal 1
BEGIN;
UPDATE lock_demo SET val = 11 WHERE id = 1;
-- Hold this, do NOT commit

-- Terminal 2
BEGIN;
UPDATE lock_demo SET val = 55 WHERE id = 5;
-- Hold this, do NOT commit

-- Terminal 1
UPDATE lock_demo SET val = 55 WHERE id = 5;
-- Blocks, waiting for Terminal 2's lock on id=5

-- Terminal 2
UPDATE lock_demo SET val = 11 WHERE id = 1;
-- MySQL detects the cycle and kills one transaction with ER_LOCK_DEADLOCK

-- Read the deadlock report
SHOW ENGINE INNODB STATUS\G
-- Find: "LATEST DETECTED DEADLOCK" section
-- It shows both transactions, what they held, what they waited for
```

**Exercise 2.5 — Understand redo log durability settings**
```sql
-- Check current setting (your lab has the safe default: 1)
SELECT variable_name, variable_value
FROM performance_schema.global_variables
WHERE variable_name = 'innodb_flush_log_at_trx_commit';
-- 1 = flush to disk on every COMMIT (safe, slower)
-- 2 = write to OS cache on COMMIT, flush every second (OS crash = data loss)
-- 0 = write to InnoDB buffer only, flush every second (MySQL crash = data loss)

-- Watch redo log writes accumulate
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN ('Innodb_os_log_written', 'Innodb_log_waits');
```

**Break-and-Diagnose: Find what's blocking a query**
```sql
-- Terminal 1: acquire a lock and hold it
BEGIN;
UPDATE lock_demo SET val = 999 WHERE id = 10;

-- Terminal 2: query that will block
UPDATE lock_demo SET val = 1000 WHERE id = 10;
-- Appears to hang...

-- Terminal 3: diagnose who is blocking whom
SELECT
  r.trx_id                            AS waiting_trx,
  r.trx_query                         AS waiting_query,
  b.trx_id                            AS blocking_trx,
  b.trx_query                         AS blocking_query,
  b.trx_started                       AS blocking_started,
  TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS blocking_for_sec
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_engine_transaction_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_engine_transaction_id;

-- Also useful:
SHOW PROCESSLIST;
-- State column will show "Waiting for this lock to be granted"
```

---

## Phase 3 — Indexes: Structure, Types, and Internals

**Goal**: Know exactly what an index is on disk and be able to predict whether any given query will use one.

**Why third**: With storage (Phase 1) and MVCC (Phase 2) in mind, you now understand that the clustered index leaf holds a versioned row with undo log pointers, and a secondary index leaf holds a primary key value plus a bookmark to find that row version. This is the complete picture. Studying indexes before MVCC gives you only half of it.

### Data Seeding

Before any exercises, create and seed the `orders` table used throughout all remaining phases.

```sql
USE lab;
CREATE TABLE orders (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id     INT           NOT NULL,
  status      TINYINT       NOT NULL,   -- 0=pending,1=paid,2=shipped,3=cancelled,4=returned
  amount      DECIMAL(10,2) NOT NULL,
  created_at  DATETIME      NOT NULL
);

-- Fast deterministic seeding: 1000x1000 cross join = 1M rows, take 500k
CREATE TABLE nums (n INT PRIMARY KEY);
INSERT INTO nums (n)
  WITH RECURSIVE r AS (SELECT 1 AS n UNION ALL SELECT n+1 FROM r WHERE n < 1000)
  SELECT n FROM r;

INSERT INTO orders (user_id, status, amount, created_at)
SELECT
  FLOOR(1 + RAND(n) * 10000),
  FLOOR(RAND(n*7) * 5),
  ROUND(10 + RAND(n*13) * 990, 2),
  NOW() - INTERVAL FLOOR(RAND(n*17) * 730) DAY
FROM nums a CROSS JOIN nums b
LIMIT 500000;

SELECT COUNT(*) FROM orders;  -- should be 500000
```

### Topics
- **B+Tree anatomy**: internal nodes (store key + child page pointer), leaf nodes (store key + data), leaf page linked list for range scans
- **Clustered index**: the table IS the B+Tree — physical row order matches primary key order, leaf = actual row data
- **Secondary indexes**: leaf nodes hold the indexed column(s) + primary key, NOT the row — double-lookup cost
- **Covering index**: when all needed columns are in the secondary index leaf — eliminates the double-lookup
- **Composite indexes**: left-prefix rule in depth, column order strategy, equality before range
- **Index selectivity and cardinality**: `SHOW INDEX FROM t`, `CARDINALITY` column, why it matters
- **Index Merge**: `Using union`, `Using intersect` — a sign of a missing composite index
- **Index Skip Scan** (MySQL 8.0.13+): optimizer reuses an index even without leftmost prefix
- **Prefix indexes**: indexing first N bytes of a string — trade-off with covering queries
- **Functional indexes**: `INDEX ((YEAR(created_at)))`, `INDEX ((LOWER(email)))`
- **Invisible indexes**: test impact of dropping an index without actually dropping it
- **Collations and index usage**: mismatched collations between joined columns prevent index use
- **NULL in indexes**: secondary indexes DO store NULLs; some operations skip them

### Key Questions to Answer
1. You have `INDEX(user_id, status)`. Will `WHERE status = 1` use this index? What about `WHERE user_id = 42`? Why?
2. The query `WHERE user_id = 42 AND status > 1 ORDER BY status` — does order of columns in a composite index matter here?
3. `EXPLAIN` shows `Using index` vs `Using index condition` — what is the difference?
4. You see `Using index merge (union)` in EXPLAIN. What does that tell you about the schema?

### Lab Exercises

**Exercise 3.1 — See the clustered index in action**
```sql
-- No index on user_id yet. Full scan with sequential leaf page reads.
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- type: ALL, rows: ~500000 — reading every leaf page of the clustered index

-- After adding an index, MySQL navigates the secondary index B+Tree first,
-- then follows primary key pointers back to the clustered index
ALTER TABLE orders ADD INDEX idx_user (user_id);
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- type: ref — used secondary index, then followed PKs to get full rows

-- Covering index: no trip back to clustered index
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
EXPLAIN SELECT user_id, status FROM orders WHERE user_id = 42;
-- Extra: Using index — all data was in the secondary index leaf, no double-lookup
```

**Exercise 3.2 — Composite index column order**
```sql
ALTER TABLE orders ADD INDEX idx_covering (user_id, status, amount, created_at);

-- Equality + range: equality columns first
EXPLAIN SELECT amount, created_at
FROM orders
WHERE user_id = 42    -- equality: can use index to narrow
  AND status IN (1,2) -- equality on next column
  AND created_at > NOW() - INTERVAL 30 DAY;  -- range: stops prefix
-- key_len tells you how many bytes of the index were used

-- Range before equality: index stops at the range column
EXPLAIN SELECT * FROM orders
WHERE created_at > NOW() - INTERVAL 30 DAY   -- range first — index almost useless here
  AND status = 1;
-- Compare rows estimate vs previous query
```

**Exercise 3.3 — Index Merge signal**
```sql
ALTER TABLE orders ADD INDEX idx_status (status);

-- Query that might trigger index merge
EXPLAIN SELECT * FROM orders WHERE user_id = 42 OR status = 1;
-- If you see: Extra: Using union(idx_user,idx_status) — two indexes merged
-- This is often slower than a full scan; it signals a missing composite index
-- Fix: for OR queries, evaluate if a single covering index or UNION rewrite is better

SET optimizer_switch = 'index_merge=off';
EXPLAIN SELECT * FROM orders WHERE user_id = 42 OR status = 1;
-- Now see what happens without merge — compare costs
SET optimizer_switch = 'index_merge=default';
```

**Exercise 3.4 — Functional index for expression queries**
```sql
-- This query cannot use an index on created_at
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- type: ALL — function wrapping the column defeats the index

-- Create a functional index
ALTER TABLE orders ADD INDEX idx_year_created ((YEAR(created_at)));
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2024;
-- type: ref — optimizer recognizes the expression matches the index
```

**Exercise 3.5 — Invisible index: test dropping safely**
```sql
-- Make an index invisible (optimizer ignores it, but it stays maintained)
ALTER TABLE orders ALTER INDEX idx_user INVISIBLE;
EXPLAIN SELECT amount FROM orders WHERE user_id = 42;
-- Optimizer no longer uses idx_user — validates impact before a real DROP

-- Restore
ALTER TABLE orders ALTER INDEX idx_user VISIBLE;
```

**Exercise 3.6 — Collation mismatch breaks index use**
```sql
CREATE TABLE users (
  id      INT PRIMARY KEY,
  email   VARCHAR(200) COLLATE utf8mb4_unicode_ci,
  INDEX idx_email (email)
);
INSERT INTO users VALUES (1,'Alice@Example.com'),(2,'bob@example.com');

-- Same collation — index is used
EXPLAIN SELECT * FROM users WHERE email = 'bob@example.com';

-- Mismatch: literal is treated as utf8mb4_0900_ai_ci (MySQL 8 default) but column is unicode_ci
-- In MySQL 8.0.28+, this can still use the index due to coercion improvements
-- But in JOINs, a mismatch between two tables' columns causes full scans:
CREATE TABLE sessions (
  id      INT PRIMARY KEY,
  email   VARCHAR(200) COLLATE utf8mb4_0900_ai_ci,  -- different collation
  INDEX idx_email (email)
);
EXPLAIN SELECT u.id FROM users u JOIN sessions s ON u.email = s.email;
-- Look for: "Using join buffer" or type:ALL on one side — collation mismatch prevented index join
```

**Break-and-Diagnose: Find unused and missing indexes**
```sql
-- Indexes that have never been read since last restart
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'lab'
  AND index_name IS NOT NULL
  AND count_read = 0
  AND index_name != 'PRIMARY';

-- Queries with full scans — candidates for new indexes
SELECT digest_text, count_star, no_index_used_count, no_good_index_used_count
FROM performance_schema.events_statements_summary_by_digest
WHERE no_index_used_count > 0
  AND schema_name = 'lab'
ORDER BY no_index_used_count DESC
LIMIT 10;
```

---

## Phase 4 — EXPLAIN and the Query Optimizer

**Goal**: Read any query plan fluently and understand why the optimizer made its choices — including when it's wrong.

**Why fourth**: The optimizer is a cost estimator that works on top of the index structures and storage statistics you now understand. Understanding its inputs (statistics, histograms, cost tables) explains why it sometimes makes wrong decisions.

### Topics
- `EXPLAIN` columns decoded: `type`, `key`, `key_len`, `rows`, `filtered`, `Extra`
- Access type ranking: `system` → `const` → `eq_ref` → `ref` → `range` → `index` → `ALL`
- `EXPLAIN FORMAT=JSON` — shows cost estimates per step (read_cost, eval_cost, prefix_cost)
- `EXPLAIN FORMAT=TREE` — shows the iterator execution model (MySQL 8.0.16+)
- `EXPLAIN ANALYZE` — executes the query and shows actual rows vs estimated, actual time (MySQL 8.0.18+)
- **Cost model**: `mysql.engine_cost`, `mysql.server_cost` — what MySQL thinks operations cost
- **Table statistics**: how InnoDB estimates cardinality (random page sampling), `innodb_stats_persistent_sample_pages`
- **Statistics staleness**: why estimates go wrong after large data changes, `innodb_stats_auto_recalc`, when to run `ANALYZE TABLE` manually
- **Histograms**: column-level statistics for non-indexed columns, `ANALYZE TABLE ... UPDATE HISTOGRAM`
- **Join algorithms**: Nested Loop (default), Hash Join (MySQL 8.0.18+, for equi-joins without index), BKA
- **Subquery strategies**: materialization, dependent subquery (correlated — runs once per outer row), semi-join transformation
- **Optimizer hints**: `/*+ INDEX() */`, `/*+ NO_HASH_JOIN() */`, `/*+ JOIN_ORDER() */`
- `optimizer_switch` flags and when to use them

### Key Questions to Answer
1. EXPLAIN shows `rows: 100` but `EXPLAIN ANALYZE` shows `actual rows: 490000` — why did the optimizer estimate so badly and how do you fix it?
2. When does MySQL choose a Hash Join over a Nested Loop join?
3. What does `Using filesort` mean and when does it NOT mean "sorting to disk"?
4. `Using temporary` — when is this avoidable and when is it unavoidable?

### Lab Exercises

**Exercise 4.1 — EXPLAIN output decoding**
```sql
-- Read every column deliberately
EXPLAIN SELECT user_id, SUM(amount)
FROM orders
WHERE status = 1
  AND created_at > NOW() - INTERVAL 90 DAY
GROUP BY user_id
ORDER BY SUM(amount) DESC
LIMIT 10;
-- id: step sequence
-- type: ref/range/ALL — how rows are accessed
-- key: which index was chosen (NULL = no index)
-- key_len: how many bytes of the composite index were used
-- rows: optimizer's estimate of rows examined
-- filtered: % of rows expected to pass WHERE after index lookup
-- Extra: Using index / Using filesort / Using temporary — critical flags
```

**Exercise 4.2 — FORMAT=JSON reveals cost**
```sql
EXPLAIN FORMAT=JSON
SELECT user_id, COUNT(*), SUM(amount)
FROM orders
WHERE status IN (1, 2)
GROUP BY user_id\G
-- Look for: "cost_info": {"read_cost": ..., "eval_cost": ..., "prefix_cost": ...}
-- These are the numbers the optimizer used to choose this plan
```

**Exercise 4.3 — EXPLAIN ANALYZE: catch bad estimates**
```sql
-- First, run without histogram so estimates are based on index cardinality only
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 2;
-- Look at: rows=XXXX (estimated) vs actual_rows=YYYY (real)
-- "status" has only 5 values but 500k rows — cardinality estimate is poor

-- Build a histogram
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 5 BUCKETS;

-- Now re-run
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 2;
-- estimated rows should now be much closer to actual
-- The histogram gave the optimizer the real distribution
```

**Exercise 4.4 — Statistics staleness simulation**
```sql
-- Record current cardinality estimate
SHOW INDEX FROM orders\G
-- Note the Cardinality value for idx_user

-- Delete 80% of rows to make statistics stale
DELETE FROM orders WHERE id % 5 != 0;

-- Cardinality is now stale (auto-recalc may not have fired yet)
SHOW INDEX FROM orders\G
-- Cardinality still shows ~old value

-- Force update
ANALYZE TABLE orders;
SHOW INDEX FROM orders\G
-- Now reflects the real row count

-- Re-seed for later phases
INSERT INTO orders (user_id, status, amount, created_at)
SELECT
  FLOOR(1 + RAND(n) * 10000),
  FLOOR(RAND(n*7) * 5),
  ROUND(10 + RAND(n*13) * 990, 2),
  NOW() - INTERVAL FLOOR(RAND(n*17) * 730) DAY
FROM nums a CROSS JOIN nums b
LIMIT 400000;
```

**Exercise 4.5 — Join algorithms**
```sql
CREATE TABLE users_dim (
  user_id INT PRIMARY KEY,
  name    VARCHAR(100),
  country CHAR(2)
);
INSERT INTO users_dim (user_id, name, country)
SELECT n, CONCAT('user_', n), ELT(1+FLOOR(RAND(n)*5), 'US','GB','DE','JP','AU')
FROM nums CROSS JOIN nums b LIMIT 10000;

-- Nested Loop join (index on orders.user_id helps)
EXPLAIN FORMAT=TREE
SELECT u.country, COUNT(*), SUM(o.amount)
FROM orders o
JOIN users_dim u ON o.user_id = u.user_id
WHERE o.status = 1
GROUP BY u.country;

-- Force Hash Join (disable BNL/BKA to see it clearly)
EXPLAIN FORMAT=TREE
SELECT /*+ HASH_JOIN(u) */ u.country, COUNT(*), SUM(o.amount)
FROM orders o
JOIN users_dim u ON o.user_id = u.user_id
WHERE o.status = 1
GROUP BY u.country;
-- Hash Join builds a hash table of the smaller input, then probes it
-- Good for large tables without index on join column
```

**Exercise 4.6 — Dependent subquery: the hidden N+1**
```sql
-- This runs the subquery ONCE PER ROW of the outer query
EXPLAIN
SELECT user_id,
       (SELECT COUNT(*) FROM orders o2 WHERE o2.user_id = o1.user_id AND o2.status = 1) AS paid_count
FROM (SELECT DISTINCT user_id FROM orders LIMIT 100) o1;
-- EXPLAIN shows: DEPENDENT SUBQUERY — executed 100 times here, N times in production

-- Rewrite as JOIN/GROUP BY
EXPLAIN
SELECT o1.user_id, COALESCE(agg.paid_count, 0)
FROM (SELECT DISTINCT user_id FROM orders LIMIT 100) o1
LEFT JOIN (
  SELECT user_id, COUNT(*) AS paid_count
  FROM orders WHERE status = 1 GROUP BY user_id
) agg ON agg.user_id = o1.user_id;
-- Now: DERIVED + single scan, not correlated
```

**Break-and-Diagnose: Force a bad plan, diagnose, fix it**
```sql
-- Disable statistics auto-recalc and create a skewed table
SET GLOBAL innodb_stats_auto_recalc = OFF;
CREATE TABLE skewed (id INT AUTO_INCREMENT PRIMARY KEY, category TINYINT, val INT, INDEX idx_cat(category));

-- Insert 100k rows: 99% category=1, 1% category=2
INSERT INTO skewed (category, val)
  SELECT IF(n <= 990, 1, 2), FLOOR(RAND(n)*1000)
  FROM nums CROSS JOIN nums b LIMIT 100000;

-- Optimizer hasn't seen the real distribution yet
EXPLAIN SELECT * FROM skewed WHERE category = 2;
-- May or may not use the index depending on stale stats

-- Build histogram to show the optimizer the real skew
ANALYZE TABLE skewed UPDATE HISTOGRAM ON category WITH 2 BUCKETS;
EXPLAIN SELECT * FROM skewed WHERE category = 2;
-- category=2 is only 1% of rows — index lookup is now clearly cheaper

SET GLOBAL innodb_stats_auto_recalc = ON;
```

---

## Phase 5 — Schema Design, Data Types, and Online DDL

**Goal**: Make schema decisions that stay performant at scale and understand how to change them safely in production.

**Why fifth**: Schema design mistakes compound — a wrong primary key or missing index discovered at 100M rows is expensive to fix. Online DDL determines whether fixing it costs 2 seconds or 2 hours of downtime. This phase connects all previous knowledge to real-world decisions.

### Topics

**Data Types**
- Integer sizing: `TINYINT` (1B) → `SMALLINT` (2B) → `INT` (4B) → `BIGINT` (8B) — affects index key size and page density
- `DECIMAL` vs `FLOAT/DOUBLE`: DECIMAL is exact (money), FLOAT/DOUBLE have rounding errors
- `DATETIME` vs `TIMESTAMP`: TIMESTAMP stores UTC and auto-converts on read, range only to 2038; DATETIME stores as-is
- `CHAR` vs `VARCHAR`: CHAR is fixed-width (good for fixed-size codes), VARCHAR stores variable-length
- `TEXT/BLOB`: never indexable in full — prefix index only; stores off-page for large values

**Primary Key Strategy**
- Auto-increment BIGINT: sequential, minimal fragmentation, predictable
- UUID (CHAR(36) or BINARY(16)): random insertion, high fragmentation, large index key
- UUID_v7 (MySQL 8.0.36+ via `UUID_TO_BIN(UUID(),1)`): time-ordered UUIDs — sequential like auto-increment, globally unique like UUID

**Normalization vs Denormalization**
- When to normalize: write-heavy, referential integrity critical
- When to denormalize: read-heavy analytics, avoiding expensive JOINs at scale

**Partitioning**
- `RANGE`, `LIST`, `HASH`, `KEY` — how each type prunes partitions
- The partition key must be in every unique index (including primary key)
- Partition pruning in EXPLAIN: `partitions` column
- When NOT to partition: OLTP workloads with point lookups rarely benefit; partitioning helps bulk DELETE and range scans on the partition key

**Generated Columns**
- `VIRTUAL` (computed on read, not stored) vs `STORED` (computed on write, stored on disk)
- Indexing a generated column — effectively a functional index alternative

**JSON Column**
- Stored as binary JSON (BSON-like), faster to read than TEXT
- Functional indexes on JSON paths: `INDEX ((data->>'$.user_id'))`
- When to use: semi-structured data with variable keys; when NOT to: when you need to query individual fields heavily (normalize instead)

**Online DDL Algorithms**
- `INSTANT`: adds column metadata only, no table rebuild — available for ADD COLUMN (not first/after position in all cases), changing defaults
- `INPLACE`: rebuilds in the background while allowing reads+writes; no full table copy
- `COPY`: builds a full copy of the table with write lock on the original — avoid on large tables
- Always specify `ALGORITHM=INPLACE, LOCK=NONE` to fail fast if MySQL would silently fall back to COPY

### Key Questions to Answer
1. You need a globally unique ID that works across microservices. What type do you choose and why?
2. Your `ALTER TABLE` to add a NOT NULL column with a default is taking 45 minutes. What algorithm is being used and how do you check?
3. You have a `JSON` column and need to query `WHERE data->>'$.status' = 'active'`. Will this be slow? How do you fix it?

### Lab Exercises

**Exercise 5.1 — Primary key fragmentation comparison**
```sql
-- Auto-increment: sequential inserts, pages fill completely
CREATE TABLE pk_auto (id BIGINT AUTO_INCREMENT PRIMARY KEY, pad CHAR(100));
INSERT INTO pk_auto (pad)
  SELECT REPEAT('x',100) FROM nums a CROSS JOIN nums b LIMIT 100000;

SELECT table_name, data_length, data_free,
       ROUND(data_free/(data_length+data_free)*100, 1) AS pct_free
FROM information_schema.tables
WHERE table_schema='lab' AND table_name IN ('pk_auto');

-- UUID: random inserts, ~50% page fill
CREATE TABLE pk_uuid (id BINARY(16) DEFAULT (UUID_TO_BIN(UUID())) PRIMARY KEY, pad CHAR(100));
INSERT INTO pk_uuid (pad)
  SELECT REPEAT('x',100) FROM nums a CROSS JOIN nums b LIMIT 100000;

SELECT table_name, data_length, data_free,
       ROUND(data_free/(data_length+data_free)*100, 1) AS pct_free
FROM information_schema.tables
WHERE table_schema='lab' AND table_name IN ('pk_uuid');

-- pk_uuid pct_free will be significantly higher — fragmentation from random inserts
```

**Exercise 5.2 — Online DDL algorithms**
```sql
-- INSTANT: just updates metadata, returns immediately regardless of table size
ALTER TABLE orders ADD COLUMN notes VARCHAR(500), ALGORITHM=INSTANT;
-- Returns in milliseconds on 500k rows

-- INPLACE: rebuilds in background, allows concurrent reads and writes
ALTER TABLE orders ADD INDEX idx_amount (amount), ALGORITHM=INPLACE, LOCK=NONE;
-- Takes seconds-minutes but table is readable and writable

-- Simulate COPY (what you want to AVOID on large tables):
-- COPY is required for some operations like changing column type
ALTER TABLE orders MODIFY COLUMN notes TEXT, ALGORITHM=COPY;
-- Observe: this builds a full copy — locks out writes for the duration
-- On a real 500GB table this would be hours

-- Always specify ALGORITHM to fail fast:
ALTER TABLE orders ADD COLUMN extra INT DEFAULT 0, ALGORITHM=COPY, LOCK=NONE;
-- ERROR: LOCK=NONE is incompatible with ALGORITHM=COPY
-- This is the error you WANT — it tells you the operation needs a lock before you run it in prod
```

**Exercise 5.3 — Generated column + functional index**
```sql
-- Extract year as a stored generated column for fast GROUP BY
ALTER TABLE orders ADD COLUMN order_year SMALLINT AS (YEAR(created_at)) STORED;
ALTER TABLE orders ADD INDEX idx_year (order_year);

EXPLAIN SELECT order_year, COUNT(*), SUM(amount)
FROM orders
GROUP BY order_year;
-- Uses idx_year — avoids full scan for year-based reports

-- JSON functional index
CREATE TABLE events (
  id   BIGINT AUTO_INCREMENT PRIMARY KEY,
  data JSON NOT NULL
);
INSERT INTO events (data) VALUES
  ('{"user_id": 1, "action": "login"}'),
  ('{"user_id": 2, "action": "purchase", "amount": 99.5}'),
  ('{"user_id": 1, "action": "logout"}');

-- Without index: full scan + JSON parse per row
EXPLAIN SELECT * FROM events WHERE data->>'$.user_id' = '1';

-- Add functional index on the JSON path
ALTER TABLE events ADD INDEX idx_event_user ((CAST(data->>'$.user_id' AS UNSIGNED)));
EXPLAIN SELECT * FROM events WHERE CAST(data->>'$.user_id' AS UNSIGNED) = 1;
-- Now uses the index
```

**Exercise 5.4 — Partitioning with pruning**
```sql
CREATE TABLE order_archive (
  id         BIGINT UNSIGNED AUTO_INCREMENT,
  user_id    INT NOT NULL,
  status     TINYINT NOT NULL,
  amount     DECIMAL(10,2),
  created_at DATE NOT NULL,
  PRIMARY KEY (id, created_at)   -- partition key must be in PK
) PARTITION BY RANGE (YEAR(created_at)) (
  PARTITION p2022 VALUES LESS THAN (2023),
  PARTITION p2023 VALUES LESS THAN (2024),
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);

INSERT INTO order_archive (user_id, status, amount, created_at)
  SELECT user_id, status, amount, DATE(created_at) FROM orders LIMIT 100000;

-- Partition pruning: optimizer eliminates irrelevant partitions
EXPLAIN SELECT * FROM order_archive
WHERE created_at BETWEEN '2023-01-01' AND '2023-12-31';
-- partitions column: shows only p2023

-- Fast DELETE by partition (instead of DELETE WHERE created_at < ...)
ALTER TABLE order_archive TRUNCATE PARTITION p2022;
-- Instant — drops the partition's pages, no row-by-row deletion
```

---

## Phase 6 — Replication Internals

**Goal**: Understand exactly what travels from primary to replica and how to manage, monitor, and break/fix replication.

**Prerequisite: Configure Replication**

Your containers are running but replication is not yet connected. Do this first.

```sql
-- On PRIMARY (port 33050)
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'replpass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- Check primary is producing GTIDs
SHOW BINARY LOG STATUS\G
-- Note: File and Position, and Executed_Gtid_Set
```

```sql
-- On REPLICA (port 33051)
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST     = 'mysql-primary',
  SOURCE_USER     = 'repl',
  SOURCE_PASSWORD = 'replpass',
  SOURCE_AUTO_POSITION = 1;   -- GTID-based: no need to specify file/position

START REPLICA;

SHOW REPLICA STATUS\G
-- Verify: Replica_IO_Running: Yes, Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0 (or small)
```

### Topics
- **Binary log**: events vs transactions, ROW format (each changed row), STATEMENT format (SQL text), when each applies
- **GTID** (Global Transaction ID): `server_uuid:sequence`, `gtid_executed` vs `gtid_purged`, how failover uses GTIDs
- **Replication threads**: IO thread (fetches binlog from primary), SQL thread (applies events on replica)
- **Parallel replication**: `replica_parallel_workers`, `replica_parallel_type=LOGICAL_CLOCK` — how MySQL determines what can apply in parallel
- **Replication lag**: what causes it (large single transactions, DDL, no parallel workers, write spikes), how to measure it accurately
- **Semi-synchronous replication**: primary waits for at least one replica to acknowledge before committing
- **Delayed replication**: `SOURCE_DELAY=N` — replica applies events N seconds behind primary (disaster recovery, accidental DELETE protection)
- **Common errors**: duplicate key on replica (1062), row not found (1032) — causes and resolution strategy

### Key Questions to Answer
1. A single `DELETE FROM orders` removes 2M rows. What happens to replication lag and why?
2. With `replica_parallel_workers = 8`, what determines whether two transactions can be applied in parallel?
3. You accidentally `DROP TABLE` on primary. You have a replica with `SOURCE_DELAY=3600`. What is your recovery window?

### Lab Exercises

**Exercise 6.1 — Read the binary log**
```sql
-- On primary
SHOW BINARY LOGS;
SHOW BINARY LOG STATUS\G

-- Read raw events from the first binlog file
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 30;
-- Notice: GTID_LOG_EVENT before each transaction, QUERY_LOG_EVENT or ROWS_LOG_EVENT for data

-- Insert a row and immediately read the event
USE lab;
INSERT INTO orders (user_id, status, amount, created_at) VALUES (99999, 1, 500.00, NOW());
SHOW BINARY LOG STATUS\G  -- note the new Position
SHOW BINLOG EVENTS IN 'mysql-bin.000001' FROM <position> LIMIT 5;
-- ROW format: you'll see TABLE_MAP_EVENT + WRITE_ROWS_EVENT, not the SQL text
```

```bash
# Read binlog from the container (human-readable)
docker exec mysql-primary mysqlbinlog --base64-output=DECODE-ROWS -v \
  /var/lib/mysql/mysql-bin.000001 | tail -50
```

**Exercise 6.2 — Verify replication is working**
```sql
-- On primary: create a table
CREATE TABLE repl_test (id INT PRIMARY KEY, val VARCHAR(50));
INSERT INTO repl_test VALUES (1, 'from primary');

-- On replica: verify it appeared
SHOW TABLES IN lab;
SELECT * FROM lab.repl_test;

-- Check GTID sets match
-- Primary:
SELECT @@GLOBAL.gtid_executed;
-- Replica:
SELECT @@GLOBAL.gtid_executed;
-- They should be equal (or replica is a subset if lag exists)
```

**Exercise 6.3 — Measure and cause replication lag**
```sql
-- On primary: large single transaction = large single apply on replica
BEGIN;
UPDATE orders SET status = 0 WHERE status != 0;  -- touches ~400k rows
COMMIT;

-- On replica: watch lag build
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source will spike during the large apply

-- More accurate lag measurement:
SELECT worker_id,
       LAST_APPLIED_TRANSACTION,
       APPLYING_TRANSACTION,
       LAST_APPLIED_TRANSACTION_END_APPLY_TIMESTAMP
FROM performance_schema.replication_applier_status_by_worker;
```

**Exercise 6.4 — Parallel replication**
```sql
-- On replica: enable parallel apply
STOP REPLICA;
SET GLOBAL replica_parallel_workers = 4;
SET GLOBAL replica_parallel_type = 'LOGICAL_CLOCK';
START REPLICA;

-- On primary: run concurrent small transactions (these can be parallelized)
-- Open 4 terminals, each running:
BEGIN; INSERT INTO orders (user_id,status,amount,created_at) VALUES (1,1,10,NOW()); COMMIT;
-- Logical clock groups transactions that committed while others were in-flight
-- These can be applied in parallel on the replica

-- Check worker activity
SELECT * FROM performance_schema.replication_applier_status_by_worker\G
```

**Exercise 6.5 — Delayed replica as a safety net**
```sql
-- On replica: set 5-minute delay
STOP REPLICA;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 300;
START REPLICA;
SHOW REPLICA STATUS\G
-- SQL_Delay: 300

-- On primary: simulate accidental delete
DELETE FROM repl_test WHERE id = 1;

-- On replica: you have 5 minutes before the DELETE arrives
-- Stop the SQL thread before the DELETE applies:
STOP REPLICA SQL_THREAD;
-- Now restore the row from the replica (it still has it)
SELECT * FROM lab.repl_test;  -- still there!

-- Cleanup
START REPLICA SQL_THREAD;
CHANGE REPLICATION SOURCE TO SOURCE_DELAY = 0;
```

**Break-and-Diagnose: Simulate and fix a replication error**
```sql
-- On replica: manually insert a row that will conflict with primary
STOP REPLICA;
SET GLOBAL super_read_only = OFF;
INSERT INTO lab.repl_test VALUES (5, 'manual on replica');
SET GLOBAL super_read_only = ON;
START REPLICA;

-- On primary: insert the same PK
INSERT INTO lab.repl_test VALUES (5, 'from primary');

-- Replica will now error: Duplicate entry '5' for key 'PRIMARY'
SHOW REPLICA STATUS\G
-- Last_SQL_Errno: 1062
-- Last_SQL_Error: ... Duplicate entry ...

-- Option 1: skip this one transaction (GTID mode)
STOP REPLICA;
SET GLOBAL super_read_only = OFF;
DELETE FROM lab.repl_test WHERE id = 5;
SET GLOBAL super_read_only = ON;
-- Get the failing GTID from Executed_Gtid_Set gap in SHOW REPLICA STATUS
-- Inject an empty transaction for the missing GTID:
SET GTID_NEXT = '<the missing GTID from the error>';
BEGIN; COMMIT;
SET GTID_NEXT = 'AUTOMATIC';
START REPLICA;
SHOW REPLICA STATUS\G  -- should be clean now
```

---

## Phase 7 — Performance Schema and Monitoring

**Goal**: Find bottlenecks using your built-in instrumentation and Grafana dashboards before they become incidents.

### Grafana Setup (do this first)

```
1. Open http://localhost:3000 (admin/admin)
2. Left sidebar → Connections → Data sources → Add new data source
3. Choose "Prometheus" → URL: http://prometheus:9090 → Save & Test
4. Left sidebar → Dashboards → Import
5. Import ID: 7362 (MySQL Overview) → select your Prometheus datasource → Import
6. Import ID: 7371 (MySQL Replication) for replica metrics
```

### Topics
- **Performance Schema architecture**: instruments (what is measured), consumers (where results go), setup tables (what is enabled)
- **Key statement tables**: `events_statements_summary_by_digest` — aggregate by query shape, not instance
- **Key I/O tables**: `table_io_waits_summary_by_index_usage` — reads per index
- **`sys` schema**: human-readable views on top of Performance Schema — start here for most diagnostics
- **`SHOW PROCESSLIST`**: thread states and what they mean
- **Slow query log**: `long_query_time`, `log_queries_not_using_indexes`, parsing with `pt-query-digest`
- **InnoDB metrics**: `SHOW ENGINE INNODB STATUS` sections decoded in full
- **Buffer pool hit ratio**: the single most important memory metric
- **Prometheus metrics from mysqld-exporter**: key metric names to know

### Thread States in SHOW PROCESSLIST (Critical Reference)

| State | Meaning | Action |
|---|---|---|
| `Sleep` | Connection idle | Normal |
| `Query` | Executing SQL | Normal |
| `Waiting for this lock to be granted` | InnoDB row lock wait | Check `data_lock_waits` |
| `Sorting result` | Filesort in memory | Often avoidable with index |
| `Copying to tmp table` | Building temp table | Check GROUP BY, subqueries |
| `Sending data` | Reading rows, sending to client | Could be network-limited |
| `statistics` | Optimizer computing plan | High count = stale stats |
| `waiting for handler commit` | Waiting for storage engine commit | Normal under heavy write load |

### Key Metrics to Target

| Metric | Target | Alert Threshold |
|---|---|---|
| Buffer pool hit ratio | > 99% | < 95% |
| Slow queries / sec | 0 | > 1 |
| Threads running (not connected) | < 20% of max_connections | > 50% |
| History list length (MVCC debt) | < 10,000 | > 100,000 |
| Seconds behind source | 0 | > 5 |
| Deadlocks / hour | 0 | > 5 |

### Lab Exercises

**Exercise 7.1 — Use SHOW PROCESSLIST to diagnose live activity**
```sql
-- Run a slow query in the background first
-- Terminal 1:
SELECT SQL_NO_CACHE user_id, SUM(amount) FROM orders GROUP BY user_id ORDER BY SUM(amount) DESC;

-- Terminal 2: observe its state
SHOW PROCESSLIST;
-- State will cycle through: statistics → Sending data → Sorting result

-- More detail:
SELECT id, user, host, db, command, time, state, LEFT(info, 80) AS query
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

**Exercise 7.2 — Find the slowest query patterns**
```sql
SELECT
  ROUND(avg_timer_wait / 1e12, 3)   AS avg_sec,
  ROUND(max_timer_wait / 1e12, 3)   AS max_sec,
  count_star                         AS executions,
  rows_examined_avg                  AS avg_rows_examined,
  rows_sent_avg                      AS avg_rows_sent,
  LEFT(digest_text, 120)             AS query_shape
FROM performance_schema.events_statements_summary_by_digest
WHERE schema_name = 'lab'
ORDER BY avg_timer_wait DESC
LIMIT 10;
-- High rows_examined vs rows_sent ratio = missing index or bad plan
```

**Exercise 7.3 — Buffer pool hit ratio**
```sql
SELECT
  FORMAT((1 - (pool_reads / pool_read_requests)) * 100, 4) AS hit_ratio_pct,
  pool_reads,
  pool_read_requests
FROM (
  SELECT
    SUM(IF(variable_name='Innodb_buffer_pool_reads', variable_value, 0))         AS pool_reads,
    SUM(IF(variable_name='Innodb_buffer_pool_read_requests', variable_value, 0)) AS pool_read_requests
  FROM performance_schema.global_status
  WHERE variable_name IN ('Innodb_buffer_pool_reads','Innodb_buffer_pool_read_requests')
) t;
-- < 99% means your buffer pool is too small for your working set
```

**Exercise 7.4 — sys schema shortcut diagnostics**
```sql
-- Top tables by I/O wait
SELECT * FROM sys.io_global_by_wait_by_bytes LIMIT 10;

-- Top queries by total time spent
SELECT * FROM sys.statement_analysis WHERE db = 'lab' LIMIT 10;

-- Unused indexes (candidates for DROP)
SELECT * FROM sys.schema_unused_indexes WHERE object_schema = 'lab';

-- Tables with full scans in recent queries
SELECT * FROM sys.schema_tables_with_full_table_scans WHERE object_schema = 'lab';

-- Current blocking situation (single query)
SELECT * FROM sys.innodb_lock_waits\G
```

**Exercise 7.5 — Read SHOW ENGINE INNODB STATUS systematically**
```sql
SHOW ENGINE INNODB STATUS\G
```
Work through each section:
- `BACKGROUND THREAD`: purge lag (history list length)
- `SEMAPHORES`: mutex/rw-lock waits — high values indicate contention
- `TRANSACTIONS`: active transactions, lock waits, history list
- `FILE I/O`: pending reads/writes, background I/O threads
- `BUFFER POOL AND MEMORY`: hit ratio, dirty pages, pages read/created/written
- `ROW OPERATIONS`: inserts/updates/deletes/reads per second
- `LOG`: LSN (log sequence number), checkpoint age — high checkpoint age = redo log pressure

**Exercise 7.6 — Enable slow query capture and analyze**
```sql
-- Your my.cnf already enables slow_query_log. Verify:
SELECT variable_name, variable_value
FROM performance_schema.global_variables
WHERE variable_name IN ('slow_query_log','long_query_time','slow_query_log_file');

-- Run a guaranteed-slow query
SELECT SQL_NO_CACHE * FROM orders o1 JOIN orders o2 ON o1.user_id = o2.user_id
WHERE o1.status = 1 LIMIT 100;
```
```bash
# Find the slow query log file
docker exec mysql-primary find /var/log /var/lib/mysql -name "*.log" -name "*slow*" 2>/dev/null

# Read it
docker exec mysql-primary tail -50 /var/lib/mysql/<hostname>-slow.log
```

---

## Phase 8 — Configuration Tuning

**Goal**: Understand what each key variable controls, how to measure its effect, and how to make changes safely.

**The rule**: never change a variable without measuring before and after. The lab exercises follow a strict pattern: measure baseline → change → measure again → interpret.

### Key Variables by Category

**Memory**
```ini
innodb_buffer_pool_size         # Working set cache. Target: 70-80% of RAM on dedicated servers.
                                # Most important single variable for read-heavy workloads.
innodb_buffer_pool_instances    # Reduces contention: 1 per GB, max 16.
innodb_log_buffer_size          # In-memory redo log buffer before flush. 64M-256M for write-heavy.
```

**Durability vs Throughput**
```ini
innodb_flush_log_at_trx_commit  # 1=flush to disk per COMMIT (safe, slowest writes)
                                # 2=flush to OS cache per COMMIT (OS crash = up to 1s loss)
                                # 0=flush every ~1s (MySQL crash = up to 1s loss)
sync_binlog                     # 1=fsync binlog per commit (safe), 0=OS decides (fast)
                                # For a replica that can be rebuilt: sync_binlog=0 is acceptable
innodb_doublewrite              # ON for crash safety, OFF only for ephemeral/test instances
```

**Redo Log (MySQL 8.0.30+ uses innodb_redo_log_capacity)**
```ini
innodb_redo_log_capacity        # Size of the redo log. Larger = less frequent checkpoints,
                                # better write throughput, longer crash recovery time.
                                # 1G-8G for write-heavy workloads.
```

**Replication**
```ini
replica_parallel_workers        # 4-16 for parallel apply. 0 = single-threaded (default, bad).
replica_parallel_type           # LOGICAL_CLOCK groups transactions for safe parallel apply.
binlog_expire_logs_seconds      # 604800 = 7 days. Keep enough for replica catch-up + backup.
```

**Connections**
```ini
max_connections                 # Sum of all connection pool max sizes + 10% headroom.
                                # Memory cost: ~1MB per thread at default thread_stack.
thread_cache_size               # Reuse threads instead of creating new ones. 16-64 is typical.
```

### Lab Exercises

**Exercise 8.1 — Measure current configuration drift from defaults**
```sql
-- Variables that have been changed from compiled defaults
SELECT v.variable_name, v.variable_value, i.variable_source
FROM performance_schema.global_variables v
JOIN performance_schema.variables_info i USING (variable_name)
WHERE i.variable_source NOT IN ('COMPILED', 'GLOBAL')
   OR v.variable_value != i.variable_value  -- not always reliable, use for reference
ORDER BY i.variable_source, v.variable_name;

-- Your my.cnf settings should all appear here
```

**Exercise 8.2 — Measure buffer pool pressure**
```sql
-- Baseline: hit ratio and dirty page ratio
SELECT
  FORMAT((bp_hit / bp_total) * 100, 4)    AS hit_ratio_pct,
  FORMAT((dirty / total_pages) * 100, 2)  AS dirty_pct,
  total_pages * 16 / 1024 / 1024          AS pool_size_gb
FROM (
  SELECT
    SUM(IF(variable_name='Innodb_buffer_pool_read_requests', variable_value, 0)) AS bp_hit,
    SUM(IF(variable_name='Innodb_buffer_pool_reads',         variable_value, 0)) AS bp_miss,
    SUM(IF(variable_name='Innodb_buffer_pool_reads',         variable_value, 0)) +
    SUM(IF(variable_name='Innodb_buffer_pool_read_requests', variable_value, 0)) AS bp_total,
    SUM(IF(variable_name='Innodb_buffer_pool_pages_dirty',   variable_value, 0)) AS dirty,
    SUM(IF(variable_name='Innodb_buffer_pool_pages_total',   variable_value, 0)) AS total_pages
  FROM performance_schema.global_status
  WHERE variable_name IN (
    'Innodb_buffer_pool_read_requests','Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_pages_dirty','Innodb_buffer_pool_pages_total'
  )
) t;
```

**Exercise 8.3 — Observe redo log checkpoint pressure**
```sql
-- Watch redo log fill level (MySQL 8.0.30+)
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
  'Innodb_redo_log_capacity_resized',
  'Innodb_redo_log_checkpoint_lsn',
  'Innodb_redo_log_current_lsn',
  'Innodb_buffer_pool_bytes_dirty'
);

-- High (current_lsn - checkpoint_lsn) = redo log is filling faster than checkpointing
-- This causes write stalls as InnoDB forces a checkpoint before accepting more writes
```

**Exercise 8.4 — Change a variable and measure the effect**
```sql
-- Record baseline write throughput
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN ('Com_insert', 'Com_update', 'Uptime');

-- Change durability setting (safe for lab, not for production without understanding)
SET GLOBAL innodb_flush_log_at_trx_commit = 2;  -- flush to OS cache, not disk, per commit

-- Run a write benchmark
DELIMITER $$
CREATE PROCEDURE bench_writes(IN n INT)
BEGIN
  DECLARE i INT DEFAULT 0;
  WHILE i < n DO
    INSERT INTO orders (user_id,status,amount,created_at)
    VALUES (FLOOR(RAND()*10000), FLOOR(RAND()*5), RAND()*1000, NOW());
    SET i = i + 1;
  END WHILE;
END$$
DELIMITER ;

CALL bench_writes(10000);

-- Check throughput again, compare Com_insert delta / time
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN ('Com_insert', 'Uptime');

-- Restore safe setting
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

**Exercise 8.5 — Connection thread monitoring**
```sql
-- Current connection state
SELECT variable_name, variable_value
FROM performance_schema.global_status
WHERE variable_name IN (
  'Threads_connected',
  'Threads_running',
  'Threads_cached',
  'Connection_errors_max_connections',
  'Max_used_connections'
);

-- If Threads_running is frequently > 10% of max_connections, you have a saturation problem
-- If Max_used_connections is close to max_connections, increase max_connections or add pooling
```

---

## Study Schedule

```
Week 1-2   →  Phase 1: Storage internals + ibd2sdi
Week 3-4   →  Phase 2: MVCC, locking, deadlocks  (budget extra time here)
Week 5-6   →  Phase 3: Indexes
Week 7-8   →  Phase 4: EXPLAIN and the optimizer
Week 9     →  Phase 5: Schema design + Online DDL
Week 10    →  Phase 6: Replication (set up first, then experiment)
Week 11    →  Phase 7: Monitoring + Grafana dashboards
Week 12+   →  Phase 8: Tuning + revisit earlier phases with new depth
```

Budget 2 weeks for Phase 2 (MVCC/locking) — this is where most engineers spend years without fully internalizing it. Every subsequent phase benefits from solid Phase 2 knowledge.

---

## Supplemental Resources

| Resource | What It Teaches |
|---|---|
| *High Performance MySQL, 4th ed.* — Schwartz, Tkachenko | Battle-tested tuning, schema design, war stories |
| MySQL 8.0 Reference Manual: *InnoDB Architecture* | Authoritative source on every internal |
| Percona blog (percona.com/blog) | Real production incidents, parameter tuning |
| Jeremy Cole's `innodb_ruby` gem | Inspect `.ibd` pages at the byte level |
| Percona Toolkit: `pt-query-digest` | Aggregate slow query logs into ranked reports |
| Percona Toolkit: `pt-online-schema-change` | Safe ALTER TABLE on production tables |

---

## Quick Reference: Lab Endpoints

| Service | Host | Port | Credentials |
|---|---|---|---|
| MySQL Primary | 127.0.0.1 | 33050 | root / root |
| MySQL Replica | 127.0.0.1 | 33051 | root / root |
| Prometheus | localhost | 9090 | — |
| Grafana | localhost | 3000 | admin / admin |
| mysqld-exporter primary | localhost | 9104 | — |
| mysqld-exporter replica | localhost | 9105 | — |
