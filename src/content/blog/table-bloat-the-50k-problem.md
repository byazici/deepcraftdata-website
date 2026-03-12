---
title: "Table Bloat: The $50K Problem Nobody Talks About"
date: 2026-03-12
summary: "Your tables are growing, your queries are slowing down, but you haven't changed anything. The culprit is probably bloat — and it's been quietly running up your bill for months."
tags: ["bloat", "vacuum", "cost", "performance"]
---

Your disk usage is climbing. Your queries are getting slower. You haven't deployed anything significant in weeks. The natural response is to reach for a bigger instance.

Before you do that — check your dead tuple counts.

There's a good chance a significant portion of your storage is occupied by data your database deleted months ago. PostgreSQL just hasn't told you about it.

## Why PostgreSQL works this way: MVCC in 60 seconds

Bloat isn't a bug. It's a deliberate trade-off baked into how PostgreSQL handles concurrency.

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**. The core idea: instead of locking a row when you update it, PostgreSQL writes a *new version* of the row and marks the old one as invisible to future transactions. Readers never block writers. Writers never block readers. This is why PostgreSQL handles concurrent load so well.

The catch: those old row versions don't disappear on their own. When you run `UPDATE`, the old row sits there. When you run `DELETE`, the row is marked as deleted but still physically occupies space. Over time, these dead tuples accumulate — and that accumulation is bloat.

On a quiet database, this is barely noticeable. On a high-write table, it's a slow-moving disaster.

## What VACUUM actually does

`VACUUM` is PostgreSQL's cleanup process. It scans tables, finds dead tuples, and marks their space as reusable. The key word is *reusable* — not *returned*.

Regular `VACUUM` does **not** shrink your table. It does not give disk space back to the operating system. It marks the space as available for future inserts and updates within PostgreSQL. Your `pg_database_size()` won't drop after a `VACUUM`. The file on disk stays the same size.

`VACUUM FULL` *does* reclaim disk space — it rewrites the entire table into a new file, leaving only live rows. But it holds an exclusive lock on the table for the entire duration. On a large production table, that can mean minutes of downtime. It's a tool for maintenance windows, not regular operations.

**Autovacuum** is the background daemon that runs `VACUUM` automatically. It's enabled by default and works well — until it doesn't. More on that shortly.

```sql
-- See when your tables were last vacuumed and how many dead tuples they have
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

If `last_autovacuum` is NULL or weeks old on a table with millions of rows, you have a problem.

## The two-headed beast: table bloat and index bloat

Most people, when they think about bloat, think about tables. They forget about indexes — which is a mistake, because **index bloat is often worse**.

Here's why: PostgreSQL's B-tree indexes grow as rows are inserted and updated, but they don't automatically shrink when rows are deleted. The tree structure gets sparser over time. On a table with high update and delete rates, an index can end up occupying two or three times the space it actually needs.

This matters for two reasons:

**Storage cost.** You're paying for all of it.

**Query cost.** A bloated index means more pages to read during index scans. More pages = more I/O = slower queries = pressure to add memory to cache it = bigger instance.

The two types of bloat feed each other. A bloated table causes more I/O, which causes more autovacuum work, which (if autovacuum can't keep up) causes more bloat.

```sql
-- Rough index bloat estimate
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

Cross-reference index size with `idx_scan`. A large index with low scan count is a double problem: it's bloated *and* possibly unused.

## The TOAST trap

PostgreSQL has a mechanism for storing large values — columns that exceed about 2KB get moved to a separate **TOAST table** (The Oversized-Attribute Storage Technique). This includes JSON blobs, large text fields, bytea columns, and anything else that doesn't fit neatly in a regular row.

Here's what most people don't realize: **TOAST tables are separate tables with separate bloat**.

If you're storing large JSON objects and updating them frequently, your TOAST table is accumulating dead tuples just like your main table. Autovacuum handles TOAST tables separately, and with default settings, it often lags behind.

The problem is invisible. Your monitoring shows the main table stats. The TOAST table sits in `pg_toast` schema, quietly consuming gigabytes, and nobody checks it.

```sql
-- Find TOAST tables and their sizes
SELECT
  n.nspname AS schema,
  c.relname AS main_table,
  t.relname AS toast_table,
  pg_size_pretty(pg_relation_size(t.oid)) AS toast_size,
  pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_class t ON c.reltoastrelid = t.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND pg_relation_size(t.oid) > 10 * 1024 * 1024  -- > 10MB TOAST tables
ORDER BY pg_relation_size(t.oid) DESC;
```

If you have tables with JSON or text columns and you've never looked at this query, you might be in for a surprise.

## The nuclear option: transaction ID wraparound

This is the part that keeps DBAs up at night — and that most application developers have never heard of.

PostgreSQL assigns a **transaction ID (XID)** to every transaction. XIDs are 32-bit integers, which means they wrap around at about 2.1 billion. PostgreSQL needs VACUUM to run regularly to "freeze" old row versions and prevent this wraparound from becoming a problem.

If VACUUM falls too far behind — if autovacuum is disabled, misconfigured, or simply overwhelmed — the database approaches the wraparound limit. When it gets close enough, **PostgreSQL shuts down and refuses to accept any writes**. The only way out is a manual `VACUUM FREEZE` on every table, which can take hours on a large database.

This is not a theoretical risk. In 2019, Cloudflare experienced a production incident where autovacuum falling behind on a high-write table triggered the wraparound protection and took down a critical service for hours.

```sql
-- Check how close you are to XID wraparound (alert if age > 1.5 billion)
SELECT
  datname,
  age(datfrozenxid) AS xid_age,
  pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

```sql
-- Per-table XID age — find the tables that need attention most urgently
SELECT
  schemaname,
  relname,
  age(relfrozenxid) AS xid_age,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||relname)) AS size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

An `xid_age` above 1.5 billion deserves immediate attention. Above 1.9 billion, you're in emergency territory.

Bloat is a cost problem. XID wraparound is an availability problem. They share the same root cause.

## How big can it actually get?

Let's make this concrete. Take a typical SaaS application table: an `events` or `activity_log` table that receives a few hundred writes per second, with rows updated and soft-deleted regularly.

With default autovacuum settings:

- Autovacuum triggers when dead tuples exceed **20% of the table** (`autovacuum_vacuum_scale_factor = 0.2`)
- On a table that grows quickly, this threshold keeps rising
- Each autovacuum run takes longer, falling further behind
- Meanwhile, the table keeps accumulating bloat between runs

After 6 months without tuning:

| Table state | Naive assumption | Reality with bloat |
|---|---|---|
| Live rows | 50M rows / ~40GB | 50M rows / ~40GB |
| Dead tuples | 0 | 15M+ rows / ~12GB |
| Index size | ~15GB | ~28GB (bloated B-trees) |
| TOAST size | ~5GB | ~9GB |
| **Total** | **~60GB** | **~89GB** |

That's roughly 50% more storage than necessary — and the queries are slower because they're scanning through all of it.

## The mistakes people make

**Disabling autovacuum.** It happens more than you'd think — usually on a "temporary" basis to speed up a bulk operation, then never re-enabled. This is the fastest path to wraparound.

**Running `VACUUM FULL` in production without a plan.** Someone discovers bloat, runs `VACUUM FULL` on a 200GB table during business hours, and holds an exclusive lock for 45 minutes. The fix causes more downtime than the problem.

**Treating instance upgrades as the fix.** Bloat makes your database *look* like it needs more resources. Bigger instance, same bloat, slightly faster degradation.

**Monitoring table sizes but not dead tuple counts.** Your disk alerts won't tell you the *composition* of what's on disk. A table that's "only" 80GB might be 30GB of live data and 50GB of garbage.

**Ignoring autovacuum logs.** When autovacuum is struggling, it logs it — but almost nobody has autovacuum alerting set up.

## The $50K math

Let's run actual numbers. Assume a mid-size production database on AWS RDS:

**Setup:**
- RDS PostgreSQL `db.r6g.2xlarge` — 64GB RAM, 8 vCPU
- 800GB gp3 storage provisioned
- Multi-AZ enabled

**Monthly costs (approximate):**
- Instance: ~$750/mo
- Storage (800GB gp3): ~$96/mo
- Multi-AZ doubles instance cost: effectively ~$1,500/mo total

**The bloat situation:**
- After 18 months with no autovacuum tuning, ~35% of storage is bloat
- That's ~280GB of dead tuples and bloated indexes
- The team upsized from `xlarge` to `2xlarge` six months ago because queries were "slow"

**What proper vacuuming and autovacuum tuning would have changed:**
- Storage: 800GB → ~520GB (-280GB × $0.12/GB = **-$33/mo**)
- The `2xlarge` was unnecessary — bloat was causing excess I/O and memory pressure. Dropping to `xlarge` + Multi-AZ: **-$750/mo**
- Plus the engineer time spent investigating "random" slowdowns: 2-3 days/quarter

**Annual delta: ~$9,500 in direct infrastructure, significantly more if you count engineer time.**

Scale that to a larger database — 3TB, multiple high-write tables, a team that's been "meaning to look at vacuum settings" for two years — and $50K is conservative.

## What to watch

You don't need to set up anything complex. These four checks cover the basics:

```sql
-- 1. Dead tuple ratio per table
SELECT relname, n_dead_tup, n_live_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;

-- 2. Tables autovacuum hasn't touched recently
SELECT relname, last_autovacuum, n_dead_tup
FROM pg_stat_user_tables
WHERE (last_autovacuum < now() - interval '7 days' OR last_autovacuum IS NULL)
  AND n_live_tup > 50000
ORDER BY n_dead_tup DESC;

-- 3. XID age — wraparound proximity
SELECT relname, age(relfrozenxid) AS xid_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC LIMIT 10;

-- 4. Autovacuum currently running (good sign if you see it regularly)
SELECT pid, relid::regclass, phase, heap_blks_scanned, heap_blks_vacuumed
FROM pg_stat_progress_vacuum;
```

Set up a weekly cron or monitoring alert on dead_pct > 20% and xid_age > 500M. That's enough to catch problems before they become expensive.

## The fix ladder

Not all bloat problems need the same solution. Work through this in order:

**1. Standard VACUUM / VACUUM ANALYZE**
For tables with moderate bloat. Fast, non-blocking, reclaims space for reuse (not for OS).

```sql
VACUUM ANALYZE your_table;
```

**2. Tune autovacuum per table**
For high-write tables, the default thresholds are too conservative. Override them:

```sql
ALTER TABLE your_high_write_table SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- trigger at 1% dead tuples, not 20%
  autovacuum_vacuum_cost_delay = 2,         -- be more aggressive
  autovacuum_analyze_scale_factor = 0.005
);
```

**3. pg_repack**
When you need to reclaim disk space *without* the locking penalty of `VACUUM FULL`. It rewrites the table in the background and swaps it in. Works on tables and indexes. This is the production-safe alternative.

```bash
pg_repack -d your_database -t your_bloated_table
```

**4. VACUUM FULL**
Last resort, maintenance window only. Reclaims the most space but requires an exclusive lock for the entire duration.

```sql
VACUUM FULL your_table;  -- plan for this taking a while on large tables
```

The right tool depends on your bloat severity, table size, and tolerance for downtime. But step 2 — per-table autovacuum tuning — is almost always the highest-leverage, lowest-risk change you can make.

---

Bloat is boring. It doesn't throw exceptions, it doesn't page you at 3am, it doesn't show up in your error logs. It just quietly makes everything a little slower and a little more expensive, month after month.

DeepCraft Audit tracks dead tuple ratios, autovacuum lag, XID age, and TOAST table growth — the metrics that most monitoring setups miss. If you'd like to see where your database stands, [run a free audit](/).
