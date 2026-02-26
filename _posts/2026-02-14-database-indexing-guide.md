---
layout: post
title: "Database Indexing: What Every Developer Should Know"
date: 2026-02-14
categories: [Databases, Performance]
---

Slow queries are one of the most common performance problems I've seen in production systems — and in the vast majority of cases, proper indexing is the fix. Here's what you actually need to understand to use indexes effectively.

## What an Index Actually Does

An index is a separate data structure (typically a B-tree) that the database maintains to speed up lookups. Think of it like the index at the back of a textbook: instead of reading every page to find a topic, you jump straight to the right page number.

Without an index, the database does a **full table scan** — reading every row to find the ones matching your condition. With an index, it walks a balanced tree to find matching rows in O(log n) time.

The trade-off: indexes consume disk space and slow down writes, because every `INSERT`, `UPDATE`, and `DELETE` must also update the index.

## The Cardinality Rule

Index columns with **high cardinality** — meaning many distinct values. A column like `user_id` (millions of distinct values) benefits enormously from an index. A column like `is_active` (only `true`/`false`) usually doesn't, because even with the index, you're still retrieving half the table.

## Composite Indexes and Column Order

A composite index on `(A, B, C)` can answer queries on:
- `A` alone
- `A` and `B`
- `A`, `B`, and `C`

But **not** on `B` alone or `C` alone. This is called the **left-prefix rule**. Order matters.

The general guidance: put the most selective column first, then match the order your queries actually filter on.

```sql
-- This query benefits from an index on (customer_id, order_date)
SELECT * FROM orders
WHERE customer_id = 123
  AND order_date > '2025-01-01';

-- An index on (order_date, customer_id) would NOT help the above query efficiently
```

## Covering Indexes

A covering index includes all the columns a query needs, so the database never has to look up the actual row. This can dramatically cut I/O for read-heavy queries:

```sql
-- If we query only these columns frequently:
SELECT user_id, email, created_at FROM users WHERE email = 'user@example.com';

-- A covering index on (email, user_id, created_at) means no row lookup at all
CREATE INDEX idx_users_email_covering ON users(email, user_id, created_at);
```

## EXPLAIN is Your Best Friend

Never guess — measure. Use `EXPLAIN` (or `EXPLAIN ANALYZE` in PostgreSQL) to see what the query planner actually does:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;
```

Look for:
- **Seq Scan** — a full table scan; usually means a missing index
- **Index Scan** — using an index to find rows, then fetching the actual row data
- **Index Only Scan** — a covering index, fastest for reads

## Common Indexing Mistakes

**1. Indexing everything**
Too many indexes slow down writes and confuse the query planner. Index what your actual slow queries need, not everything speculatively.

**2. Functions on indexed columns**
```sql
-- This can't use an index on `created_at`:
WHERE YEAR(created_at) = 2025

-- This can:
WHERE created_at BETWEEN '2025-01-01' AND '2025-12-31'
```

**3. Ignoring index bloat**
After many updates and deletes, indexes develop dead pages (especially in PostgreSQL). Periodic `VACUUM` and `REINDEX` keep them lean.

**4. Not monitoring unused indexes**
Every index costs write performance. Most databases track index usage stats — drop indexes that haven't been used in months.

## Takeaway

Good indexing is a skill developed through measurement, not intuition. Profile your slow queries with `EXPLAIN`, understand cardinality and the left-prefix rule, and build only the indexes your workload actually needs. A handful of well-chosen indexes beats dozens of speculative ones every time.
