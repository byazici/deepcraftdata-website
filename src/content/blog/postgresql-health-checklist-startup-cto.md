---
title: "PostgreSQL Health Checklist for Startup CTOs"
date: 2026-03-13
summary: "Your database has been in production for a while. It still works — but something's off. This checklist covers what to check before you reach for a bigger instance."
tags: ["performance", "checklist", "production", "audit"]
---

This isn't a guide for the first week of production. It's for when your database has been running for 18+ months, your team has shipped a few hundred features, and queries that were instant at launch now occasionally pause. The database still works. But you've started noticing things.

This checklist is structured around what actually fails in practice — not textbook categories. Each item includes diagnostic queries you can run right now.

---

## 1. Autovacuum is keeping up with your write load

This is almost always the first thing to check and the last thing people think of.

PostgreSQL's MVCC model means every UPDATE writes a new row version and leaves the old one as a dead tuple. VACUUM cleans these up. Autovacuum is the background process that runs VACUUM automatically — but with default settings, it's calibrated for a database that isn't doing much work. At any meaningful write load, it falls behind.

```sql
-- Dead tuple ratio per table — anything above 10-15% on large tables is a problem
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_live_tup > 50000
ORDER BY dead_pct DESC NULLS LAST
LIMIT 20;
```

**What to look for:** `dead_pct` above 15% on any table over 100k rows. `last_autovacuum` being NULL or weeks old. Both indicate autovacuum isn't keeping up.

For high-write tables, the default `autovacuum_vacuum_scale_factor = 0.2` (trigger at 20% dead tuples) is too conservative. Override it per table:

```sql
ALTER TABLE your_high_write_table SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.005,
  autovacuum_vacuum_cost_delay = 2
);
```

Also check XID age. This is the metric that turns a performance problem into an availability incident:

```sql
-- Alert threshold: xid_age > 1.5 billion. Emergency: > 1.9 billion.
SELECT
  datname,
  age(datfrozenxid) AS xid_age,
  pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

If you're above 1.5 billion, schedule a `VACUUM FREEZE` pass on your oldest tables. Don't wait.

---

## 2. Indexes are earning their write cost

Every index on a table is updated on every INSERT, UPDATE, and DELETE. If that index is never used for queries, it's pure overhead.

After a year or two of feature development — schema migrations, deprecated ORM models, refactored query patterns — most databases have accumulated indexes that serve no one.

```sql
-- Indexes that haven't been scanned since last statistics reset
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%pkey%'
  AND indexrelname NOT LIKE '%unique%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Caveat:** Statistics reset on restart. If your database restarted recently, these results are misleading. Cross-reference with `pg_stat_reset_shared('bgwriter')` to see when stats were last reset, or wait until your instance has been running for at least a few weeks.

Dropping a 2GB unused index on a write-heavy table is one of the highest-leverage, lowest-risk changes you can make. It reduces storage, reduces write amplification, and slightly improves insert/update throughput.

---

## 3. Sequential scans aren't eating your I/O budget

A sequential scan on a 5GB table reads every page of that table into memory. If it's displacing useful cached pages in `shared_buffers`, it's wrecking the cache efficiency for everything else. If it's happening on every request, you're bleeding I/O.

```sql
-- Tables with the most sequential scans — high seq_scan + high seq_tup_read is the problem
SELECT
  schemaname,
  relname,
  seq_scan,
  seq_tup_read,
  idx_scan,
  pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS table_size,
  round(seq_scan::numeric / nullif(seq_scan + idx_scan, 0) * 100, 1) AS seq_pct
FROM pg_stat_user_tables
WHERE pg_relation_size(schemaname||'.'||relname) > 100 * 1024 * 1024  -- > 100MB
ORDER BY seq_tup_read DESC
LIMIT 15;
```

**What to look for:** Large tables (>100MB) where `seq_pct` is high and `seq_scan` is in the thousands. This indicates queries that are either missing indexes or have query plans that aren't using them.

For the specific queries, you need `pg_stat_statements`:

```sql
-- Top queries by total I/O — requires pg_stat_statements extension
SELECT
  round(total_exec_time::numeric, 0) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(blk_read_time + blk_write_time, 0) AS total_io_ms,
  left(query, 100) AS query_preview
FROM pg_stat_statements
WHERE calls > 100
ORDER BY blk_read_time + blk_write_time DESC
LIMIT 20;
```

If `pg_stat_statements` isn't enabled, that's item 9 on this list. Enable it — the data is irreplaceable.

---

## 4. Memory configuration reflects your actual workload

The defaults PostgreSQL ships with are tuned for a machine with 1GB of RAM and a workload from 2000. They're wildly inappropriate for a production database in 2025.

The three settings that matter most:

**`shared_buffers`** — PostgreSQL's own buffer cache. The canonical advice is 25% of RAM. That's correct as a starting point, but the right value depends on your working set size. On a 64GB instance, 16GB is reasonable. On a 32GB instance, 8GB is reasonable.

**`effective_cache_size`** — This doesn't allocate memory. It tells the query planner how much memory it can assume the OS has available for caching. If it's set too low, the planner underestimates the cost of index scans and chooses sequential scans instead. It should be set to 50-75% of total RAM.

**`work_mem`** — Allocated per sort operation, not per connection. One complex query with multiple hash joins can use several multiples of `work_mem`. On a database with 50 active connections, setting `work_mem = 256MB` can result in 12GB+ of allocation. Keep it at 16-64MB globally and bump it for specific sessions when needed.

```sql
-- Current memory settings and active connection count
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'effective_cache_size',
               'maintenance_work_mem', 'max_connections');
```

```sql
-- How many connections are actually doing something vs. idle
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state
ORDER BY count DESC;
```

If most of your connections are idle, you're paying for memory overhead on connections that aren't doing work. That's the case for a connection pooler.

---

## 5. You have a connection pooler in front of the database

PostgreSQL is not designed to handle hundreds of direct connections efficiently. Each connection spawns a backend process with its own memory allocation. At 200+ direct connections, you're paying for overhead that a connection pooler eliminates.

PgBouncer in transaction pooling mode allows thousands of application connections to multiplex over a much smaller pool of actual PostgreSQL connections. The operational cost of running PgBouncer is low. The upside on a busy application is significant — both in memory usage and in the ability to downsize your instance.

```sql
-- Check max_connections vs actual usage
SELECT
  (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections,
  count(*) AS current_connections,
  sum(CASE WHEN state = 'idle' THEN 1 ELSE 0 END) AS idle,
  sum(CASE WHEN state = 'active' THEN 1 ELSE 0 END) AS active,
  sum(CASE WHEN state = 'idle in transaction' THEN 1 ELSE 0 END) AS idle_in_tx
FROM pg_stat_activity
WHERE datname = current_database();
```

**What to watch for:** `idle_in_tx` counts above zero. These are connections that opened a transaction, did some work, and stopped — while holding locks and pins on rows. They block autovacuum, cause lock contention, and can cascade into application hangs. An `idle in transaction` connection that's been sitting for more than a few seconds usually indicates an application bug.

---

## 6. Long-running transactions aren't creating lock contention

Lock waits are invisible until they're catastrophic. A long-running transaction holds locks on the rows it touched. Any other transaction that needs those rows queues behind it. If the queue grows fast enough, you get a pile-up.

```sql
-- Long-running transactions (over 5 minutes)
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state,
  wait_event_type,
  wait_event
FROM pg_stat_activity
WHERE state != 'idle'
  AND query_start < now() - interval '5 minutes'
ORDER BY duration DESC;
```

```sql
-- Currently blocked queries and what's blocking them
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  now() - blocked.query_start AS wait_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
ORDER BY wait_duration DESC;
```

Schema migrations are a common culprit. An `ALTER TABLE` that requires a full table rewrite holds `ACCESS EXCLUSIVE` lock for its entire duration — blocking every read and write. For large tables, use tools like `pg_repack` or migration patterns that don't require full rewrites (`ADD COLUMN` with a default is safe in PostgreSQL 11+).

---

## 7. Checkpoints aren't hammering disk

PostgreSQL writes dirty pages to disk at checkpoint intervals. If checkpoints happen too frequently, you get sustained I/O spikes. If they're too infrequent, recovery after a crash takes longer.

```sql
-- Checkpoint frequency and write patterns
SELECT
  checkpoints_timed,
  checkpoints_req,
  checkpoint_write_time,
  checkpoint_sync_time,
  buffers_checkpoint,
  buffers_clean,
  buffers_backend,
  maxwritten_clean,
  stats_reset
FROM pg_stat_bgwriter;
```

**What to look for:** `checkpoints_req` significantly higher than `checkpoints_timed` means checkpoints are being forced by WAL accumulation — `max_wal_size` is too low. `maxwritten_clean > 0` frequently means the bgwriter can't keep up, and `buffers_backend` is high means the backend processes are having to write dirty pages themselves, which adds latency to queries.

Typical tuning starting points for a production database:

```
checkpoint_completion_target = 0.9
max_wal_size = 2GB          # increase if checkpoints_req >> checkpoints_timed
wal_buffers = 64MB
```

---

## 8. Your largest tables have a partitioning plan

Partitioning isn't free — it adds schema complexity, changes query planning behavior, and requires maintenance. But once a table crosses roughly 100-200GB, the operational pain of not having partitioning often exceeds the cost of adding it.

The common cases where partitioning pays off:

- **Time-series data** (events, logs, audit trails) where old data is queried rarely and you want to drop old partitions instead of running expensive DELETEs
- **Tenant-segmented data** where most queries filter by tenant ID and partition pruning eliminates entire partitions from scans
- **Tables with regular bulk deletes** — partitioned DELETEs (`DROP PARTITION`) are instant; row-level DELETEs on a 500GB table are slow and cause bloat

```sql
-- Large tables that might be partition candidates
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
  pg_total_relation_size(schemaname||'.'||tablename) AS raw_size,
  n_live_tup
FROM pg_stat_user_tables
WHERE pg_total_relation_size(schemaname||'.'||tablename) > 10 * 1024 * 1024 * 1024  -- > 10GB
ORDER BY raw_size DESC;
```

If you have a table over 50GB and it doesn't have a partitioning plan, you should at minimum have a documented reason for why it doesn't need one.

---

## 9. pg_stat_statements is enabled and you're reading it

This is the single most valuable observability extension in PostgreSQL. Without it, you're flying blind on query-level performance. With it, you can identify which queries are consuming the most cumulative time, I/O, or plan cache misses.

```sql
-- Check if it's enabled
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';
```

If it's not there:

```sql
-- Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- Then in the database:
CREATE EXTENSION pg_stat_statements;
```

Enabling it requires a restart. It's worth scheduling one.

Once enabled, the queries that matter most:

```sql
-- Top 20 queries by total execution time
SELECT
  calls,
  round(total_exec_time::numeric, 0) AS total_ms,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  rows,
  round((blk_read_time + blk_write_time)::numeric, 0) AS io_ms,
  left(query, 120) AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY total_exec_time DESC
LIMIT 20;
```

The `stddev_ms` column is underused. High standard deviation relative to mean indicates inconsistent execution — sometimes fast, sometimes very slow. This usually points to plan instability, cache misses on large tables, or lock waits.

---

## 10. You know what your top 5 tables look like over time

This isn't a query, it's a process.

At a certain database size, decisions made without historical context cause problems. You add an index to fix a slow query, but you don't know if the table it's on has been growing 10% month-over-month and the index will be 40GB in a year. You tune autovacuum thresholds, but you don't know if your write rate has doubled in the last quarter.

The minimum useful tracking:

```sql
-- Snapshot this weekly into a tracking table
SELECT
  now() AS captured_at,
  schemaname,
  relname,
  pg_total_relation_size(schemaname||'.'||relname) AS total_bytes,
  n_live_tup,
  n_dead_tup,
  seq_scan,
  idx_scan
FROM pg_stat_user_tables
ORDER BY total_bytes DESC
LIMIT 50;
```

Store this in a `db_health_snapshots` table or export it to your data warehouse. Growth rate curves are more actionable than point-in-time numbers. A table at 80GB that grew 5GB last month is a different problem from a table at 80GB that grew 5GB over two years.

---

## The triage order

If you're looking at this list for the first time and want to know where to start:

1. **XID age** — if any table is above 1.5B, handle it before anything else. This is the only item on this list that can take your database down.
2. **Dead tuple ratios** — identify which tables have bloat above 15% and tune autovacuum on them. This is high leverage and low risk.
3. **pg_stat_statements** — enable it if it's off. You need a week of data before you can act on query-level findings.
4. **Idle in transaction connections** — if you see these regularly, find and fix the application bug. It's the source of multiple downstream problems.
5. **Sequential scans on large tables** — once you have `pg_stat_statements` data, identify the queries doing full scans and decide if they need indexes.

Everything else is tuning. The first four items are about preventing incidents.

---

DeepCraft Audit checks all of this automatically — dead tuple ratios, XID age, missing and unused indexes, memory configuration, connection patterns, checkpoint health, and query-level I/O from `pg_stat_statements`. If you'd rather see a report than run queries, [run a free audit](/).
