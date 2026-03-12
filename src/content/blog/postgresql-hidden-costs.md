---
title: "Is Your PostgreSQL Silently Costing You Thousands?"
date: 2026-03-12
summary: "Your database bill keeps climbing, but your feature list hasn't changed. Here's how hidden PostgreSQL inefficiencies translate into real money — and what you can actually do about it."
tags: ["cost", "performance", "audit"]
---

You upgraded the instance six months ago. The app felt faster for a week, then settled back to "acceptable." The cloud bill went up and stayed there. Sound familiar?

Most PostgreSQL cost problems aren't dramatic. There's no single query that's destroying your budget. Instead, it's a slow accumulation of small inefficiencies — bloated tables, misconfigured memory, missing indexes — that compound quietly until you're paying 20% more than you need to. Every month.

Let's talk about where that money actually goes.

## The compounding problem: data grows, waste grows faster

When your database is small, none of this matters. A sequential scan on a 50MB table is fine. Autovacuum falling slightly behind is fine. `shared_buffers` being under-configured is fine.

But databases don't stay small. And the painful thing about PostgreSQL inefficiencies is that they scale with your data, not just your load.

A table with 10 million rows and 15% dead tuple bloat wastes disk space. The same table at 200 million rows is now wasting gigabytes — and slowing down every query that touches it, increasing I/O, increasing the memory needed to cache it, and pushing you toward a bigger (more expensive) instance to compensate.

The inefficiency doesn't just cost more at scale. It costs *disproportionately* more.

## Where the money actually goes

### Cloud compute: you're probably on the wrong instance

The most visible line on your bill is instance size. And the instinct when things get slow is to go up a tier.

Sometimes that's the right call. Often, it isn't.

On AWS RDS, going from `db.r6g.xlarge` (32GB RAM, ~$375/mo) to `db.r6g.2xlarge` (64GB RAM, ~$750/mo) doubles your spend. Before you make that jump, it's worth asking: *why* does it need more RAM?

Common answers:
- `shared_buffers` is pulling too much data into memory because of full-table scans that an index would prevent
- `work_mem` is set too high and multiplied by 50+ active connections
- Bloated tables mean more data has to be cached just to serve the same queries

Fix the root cause and you might not need the bigger instance at all.

### Storage and SSD: the bloat tax

PostgreSQL doesn't reclaim disk space automatically when you delete or update rows. Dead tuples sit there, taking up space, until autovacuum gets around to them. When autovacuum falls behind — which happens easily on high-write tables — bloat accumulates.

The cost shows up in two ways:

**Direct storage cost.** On RDS gp3, you pay per GB provisioned. If 30% of your storage is dead tuples, you're paying for data that provides zero value.

**Indirect I/O cost.** Bloated tables mean more pages to scan, more data to move through I/O, and more pressure on your buffer cache. You're not just paying for the storage — you're paying for all the work done *on* that storage.

Index bloat is equally sneaky. Indexes grow as rows are inserted and updated, but they don't shrink when rows are deleted. A heavily updated table can end up with an index twice the size it needs to be, contributing to both storage and I/O costs.

```sql
-- Quick check for table bloat
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
  pg_size_pretty(
    pg_total_relation_size(schemaname||'.'||tablename) -
    pg_relation_size(schemaname||'.'||tablename)
  ) AS index_size,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct
FROM pg_stat_user_tables
WHERE n_live_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

If `dead_pct` is consistently above 10-15% on your large tables, autovacuum isn't keeping up.

### RAM: paying for memory you're wasting

Memory is where PostgreSQL configuration mistakes get expensive fast.

The two biggest culprits:

**`shared_buffers` sized wrong.** The common advice is "set it to 25% of RAM." That's a starting point, not a rule. If your working set is much smaller than 25% of RAM, you're paying for memory that's not doing useful work. If sequential scans are pulling huge amounts of data through the cache and evicting actually useful pages, more `shared_buffers` won't help — you need better query plans.

**`work_mem` multiplied by connections.** `work_mem` is allocated *per sort operation*, not per connection. One complex query with multiple sorts can use several multiples of `work_mem`. Multiply that by 50 active connections and you've potentially allocated several gigabytes of RAM to sort buffers — RAM that you're paying for in your instance tier.

```sql
-- See how many connections are actually active
SELECT
  state,
  count(*),
  max(now() - state_change) AS longest_in_state
FROM pg_stat_activity
WHERE datname = current_database()
GROUP BY state;
```

Idle connections still hold memory. If you have 100 connections and 80 are idle, you're paying for connection overhead that a connection pooler (PgBouncer) could eliminate — often allowing you to drop to a smaller instance.

## The usual suspects

In practice, PostgreSQL cost audits keep turning up the same patterns:

**Unused indexes.** Every index you maintain has a write cost. Inserts, updates, and deletes all have to update every index on the table. If an index is never used for queries, it's pure overhead — storage + write amplification with no benefit.

```sql
-- Indexes that haven't been used since last stats reset
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Sequential scans on large tables.** One seq scan on a 10GB table can evict half your buffer cache. If it's happening regularly, you're paying for the I/O and you're paying for the cache churn.

**Autovacuum falling behind.** On high-update tables, autovacuum often can't keep up with default settings. The fix is usually tuning per-table autovacuum parameters, not scaling hardware.

**ORM-generated N+1 queries.** Not directly a cost issue, but N+1 patterns create artificial load that makes your database *look* like it needs more resources.

## What to actually observe

You don't need a fancy monitoring tool to get started. PostgreSQL's built-in statistics views tell you most of what you need:

| View | What it tells you |
|---|---|
| `pg_stat_user_tables` | Dead tuple counts, seq scan frequency, last vacuum/analyze |
| `pg_stat_user_indexes` | Index usage counts — find unused indexes |
| `pg_stat_statements` | Query-level stats: total time, call count, I/O (requires the extension) |
| `pg_stat_bgwriter` | Checkpoint and buffer write patterns |
| `pg_stat_activity` | Current connection states and query activity |

Enable `track_io_timing = on` in your config if you haven't already. It adds minimal overhead and unlocks actual I/O wait time data — which is far more useful than ratio-based metrics alone.

## The 5–20% savings reality

We're not talking about a complete rewrite or months of optimization work. A focused audit typically finds:

- **Quick wins (1-2 days):** Drop unused indexes, fix autovacuum settings on bloated tables, right-size `work_mem`. These often recover 5-10% of costs immediately.
- **Medium effort (1-2 weeks):** Add missing indexes to eliminate seq scans, introduce a connection pooler, revisit instance sizing based on actual memory usage. Another 5-10%.
- **Longer term:** Schema and query-level improvements that reduce data volume and I/O. The upper end of savings lives here.

The 5–20% range is conservative. On databases that haven't been audited in a while, we regularly see more. The point is that you don't need heroic engineering to move the needle — you need visibility into the right metrics.

## Getting started without a full rewrite

The practical approach:

1. **Run the queries above.** Dead tuple ratios, unused indexes, connection states. Take a snapshot.
2. **Enable `pg_stat_statements`** if it isn't on. A week of query-level data is more actionable than any single point-in-time check.
3. **Fix the obvious things first.** Drop unused indexes on large tables. Tune autovacuum on your top-5 most-written tables. These are low-risk changes with measurable impact.
4. **Revisit instance sizing after, not before.** Once you've addressed the obvious waste, you'll have a clearer picture of what resources you actually need.

The goal isn't a perfectly optimized database. It's a database where you're not paying for problems you don't know you have.

---

DeepCraft Audit runs these checks automatically — dead tuple ratios, unused indexes, bloat estimates, connection patterns, and query-level I/O — and surfaces what's actually worth fixing. If you want to see what your PostgreSQL is silently costing you, [give it a try](/).
