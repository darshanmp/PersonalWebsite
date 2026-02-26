---
layout: post
title: "Building Reliable Data Pipelines: Lessons from Production"
date: 2026-01-24
categories: [Data Engineering, Big Data]
---

After spending years working with data pipelines at scale, I've come to appreciate that the hard part isn't moving data — it's doing it reliably, consistently, and in a way your team can actually maintain. Here are the most valuable lessons I've learned.

## 1. Idempotency is Non-Negotiable

Your pipeline will fail. When it does, you need to safely re-run it without creating duplicate records or corrupting state. Design every step to be idempotent — running it twice should produce the same result as running it once.

The simplest approach: use upserts rather than inserts, and partition your data by a timestamp or batch ID so re-runs naturally overwrite the same partition.

```sql
-- Bad: INSERT can create duplicates on retry
INSERT INTO orders SELECT * FROM staging.orders WHERE batch_id = 42;

-- Good: MERGE/UPSERT handles retries cleanly
MERGE INTO orders AS target
USING (SELECT * FROM staging.orders WHERE batch_id = 42) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;
```

## 2. Separate Raw from Processed Data

Always land raw data first before any transformation. This "bronze layer" approach (popularized by Databricks' Medallion architecture) means:

- **Bronze**: raw, immutable copy of source data
- **Silver**: cleaned, deduplicated, validated data
- **Gold**: aggregated, business-ready tables

If your transformation logic has a bug, you can re-process from bronze without re-fetching from the source. This has saved me countless hours.

## 3. Build Observability In from Day One

You cannot fix what you cannot see. Every pipeline should expose:

- **Row counts** at each stage (source vs. destination)
- **Null rates** for critical columns
- **Processing time** per batch
- **Data freshness** — how stale is the data right now?

Set up alerts when row counts drop significantly or when a job hasn't run in the expected window. Silent failures are the worst kind.

## 4. Handle Late-Arriving Data

In real-world systems, data arrives late. Events timestamped yesterday might show up tomorrow due to mobile clients syncing offline, delayed webhooks, or upstream system lag.

Design your aggregations with a configurable **lookback window**. Instead of aggregating "yesterday's" data once at midnight, reprocess the last 3–7 days on each run. Yes, it's more compute — but it's far better than systematically under-counting.

## 5. Schema Evolution is Inevitable

Sources change. A field gets renamed, a new column appears, a type changes from `int` to `string`. Without a strategy, this breaks your pipeline silently.

Practical approaches:
- Use a schema registry (Confluent, AWS Glue) to track and enforce schemas
- Add schema validation at the ingestion step with clear, actionable error messages
- Version your transformations so you can backfill historical data with new logic

## 6. Make Failures Visible and Actionable

When a step fails, the error message should tell an on-call engineer exactly what happened and ideally what to do about it. Generic `NullPointerException` messages at 2am help no one.

Use structured logging and include contextual info:

```python
logger.error(
    "Transformation failed",
    extra={
        "batch_id": batch_id,
        "source_table": source_table,
        "record_count": record_count,
        "error": str(e),
    }
)
```

## The Takeaway

Reliable data pipelines come down to a few core principles: be idempotent, preserve raw data, instrument everything, and plan for the unexpected. The engineering isn't glamorous — but solid pipelines are the foundation that lets data teams focus on insight rather than firefighting.
