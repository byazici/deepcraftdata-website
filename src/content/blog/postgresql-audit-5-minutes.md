---
title: "How to Run a PostgreSQL Audit in 5 Minutes"
date: 2026-03-13
summary: "A PostgreSQL audit doesn't have to be a multi-day project. Here are the five queries that surface 80% of production issues — and what to do when you find them."
tags: ["audit", "performance", "checklist"]
---

"Audit" sounds like a project. Something you schedule, plan, and bring a consultant in for. In practice, the most common production PostgreSQL issues show up in five queries. Running them takes less than five minutes. Understanding the output takes a bit more.

This is the diagnostic run. Not a full optimization project — just the checks that catch the problems that actually hurt people in production.

Open a `psql` session or your preferred client. Run these in order.

---

## Query 1: Are your tables accumulating dead weight?

```sql
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum,
  age(relfrozenxid) AS xid_age
FROM pg_stat_user_tables t
JOIN pg_class c ON c.relname = t.relname
WHERE n_live_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

**What you're looking for:**

- `dead_pct` above 15% on any large table — autovacuum is falling behind
- `last_autovacuum` being NULL or more than a few days old — it's not running at all
- `xid_age` above 1 billion — you want to know before it becomes urgent

A high `dead_pct` on a high-write table (events, logs, audit trails, sessions) is normal-ish, but unattended it compounds into storage bloat, slower queries, and eventually the transaction ID wraparound problem that can halt writes entirely. PostgreSQL will warn you in logs well before that happens, but most teams don't have autovacuum alerting configured.

---

## Query 2: Are you paying for indexes nobody uses?

```sql
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
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

**What you're looking for:**

Large indexes (anything over a few hundred MB) with `idx_scan = 0`. Every index on a table is updated on every write. If an index is never read, it's write amplification and storage cost with no benefit.

**Caveat:** Statistics reset on database restart. If your instance restarted recently, this list may show indexes as unused that actually run constantly. Check `pg_stat_database.stats_reset` to see when the clock started.

Dropping a 1GB unused index on a high-write table is one of the cleanest wins in PostgreSQL optimization. Low risk, measurable impact, reversible with `CREATE INDEX CONCURRENTLY` if you made a mistake.

---

## Query 3: Are you reading entire tables when you shouldn't be?

```sql
SELECT
  schemaname,
  relname,
  seq_scan,
  idx_scan,
  pg_size_pretty(pg_relation_size(schemaname||'.'||relname)) AS table_size,
  round(seq_scan::numeric / nullif(seq_scan + idx_scan, 0) * 100, 1) AS seq_pct
FROM pg_stat_user_tables
WHERE pg_relation_size(schemaname||'.'||relname) > 50 * 1024 * 1024
ORDER BY seq_scan DESC
LIMIT 15;
```

**What you're looking for:**

Large tables (>50MB) with high `seq_scan` counts. A sequential scan on a 10GB table reads every page of that table into memory, displacing useful cached data and burning I/O. If it's happening thousands of times, a missing index is draining your performance budget.

To find which specific queries are causing these scans:

```sql
-- Requires pg_stat_statements (see below if it's not enabled)
SELECT
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((blk_read_time + blk_write_time)::numeric, 0) AS io_ms,
  left(query, 100) AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY blk_read_time + blk_write_time DESC
LIMIT 10;
```

---

## Query 4: Is your memory configuration appropriate for this instance?

```sql
SELECT
  name,
  setting,
  unit,
  CASE name
    WHEN 'shared_buffers' THEN
      pg_size_pretty(setting::bigint * 8192) || ' — should be ~25% of RAM'
    WHEN 'work_mem' THEN
      setting || unit || ' per sort op — multiply by active connections × sorts per query'
    WHEN 'effective_cache_size' THEN
      pg_size_pretty(setting::bigint * 8192) || ' — should be ~50-75% of RAM'
    WHEN 'max_connections' THEN
      setting || ' — if most are idle, consider PgBouncer'
    ELSE setting || coalesce(unit, '')
  END AS interpretation
FROM pg_settings
WHERE name IN (
  'shared_buffers', 'work_mem', 'effective_cache_size',
  'max_connections', 'checkpoint_completion_target', 'max_wal_size'
)
ORDER BY name;
```

Then check how many connections are actually doing something:

```sql
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state ORDER BY count DESC;
```

**Common findings:**

- `work_mem` at 256MB or higher on a database with 100+ connections is a memory pressure problem waiting to happen
- `effective_cache_size` left at the default 4GB on a 32GB instance causes the query planner to underestimate index usefulness and choose sequential scans
- `max_connections = 200` with 170 idle connections is an argument for a connection pooler, not a bigger instance

---

## Query 5: Is pg_stat_statements enabled?

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'pg_stat_statements';
```

If this returns nothing, you're operating without query-level performance data. You can see that *something* is slow. You can't see *which queries* are slow, how often they run, or what percentage of your total database load they account for.

Enable it:

```sql
-- In postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- Then after restart:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

This requires a database restart. It's worth scheduling one. The data you get back — total time per query, I/O time, variance, row counts — is the foundation of any serious query optimization work.

Also worth enabling if it isn't on:

```
track_io_timing = on
```

This adds near-zero overhead and unlocks actual I/O wait time data in `pg_stat_statements` and `pg_stat_database`. Without it, you're guessing at whether slowness is CPU-bound or I/O-bound.

---

## What to do with the results

Most people run these queries, see a few yellow flags, and don't know which one to deal with first. A rough priority order:

**Act immediately if:**
- `xid_age` is above 1.5 billion on any table — run `VACUUM FREEZE` on the highest-age tables
- `idle in transaction` connections are sitting for more than a few minutes — this indicates an application bug that's holding locks

**Address this week:**
- Tables with `dead_pct` above 20% — tune per-table autovacuum settings
- Unused indexes over 500MB on write-heavy tables — schedule a drop

**Address this month:**
- Sequential scans on large tables — investigate query plans, add missing indexes
- Memory settings that are clearly misconfigured for your instance size
- `pg_stat_statements` and `track_io_timing` if they're not on

---

## Where 5 minutes falls short

Running the queries is the easy part. Interpreting the output requires context you don't have from a single point-in-time snapshot:

**Is this dead_pct number new or chronic?** A table at 18% dead tuples might have spiked today after a bulk operation, or it might have been sitting there for months. Without historical data, you can't tell.

**Is this seq_scan count high relative to traffic?** 10,000 sequential scans sounds like a lot. On a table that gets 5 million hits per day, it might be fine. On a table that should only be hit by a nightly job, it's a problem.

**What's the actual cost of this index?** A 2GB unused index on a table with 50 writes per second has a measurably different cost than the same index on a table with 2 writes per hour.

**Are there issues in TOAST tables, background jobs, and replication lag?** The five queries above don't cover everything. TOAST table bloat, autovacuum worker contention, replica lag on heavy-write periods, and checkpoint pressure require additional checks.

The five-minute audit tells you where to look. A real audit tells you what you're actually looking at.

---

DeepCraft Audit runs all of these checks automatically — including TOAST tables, autovacuum worker status, query-level I/O from `pg_stat_statements`, index bloat estimates, memory configuration analysis, and connection patterns — and surfaces findings with the context needed to act on them. No access to your data, no agent to install: you connect your database and get a report. [Run a free audit](/).
