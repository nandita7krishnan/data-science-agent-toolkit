---
name: incremental-model-builder
description: Write or modify an incremental SQL/dbt model — pick the right strategy by grain/scale/platform, configure unique_key/cluster/event_time, handle late-arriving data and idempotency.
when_to_invoke:
  - "Make this model incremental"
  - "What incremental strategy should I use for X?"
  - "Modify the dbt model for <table> to be incremental"
  - "Build an incremental load for X"
---

# Incremental Model Builder

Incremental models trade simplicity for cost. Get the strategy wrong and you'll either silently miss rows (append for an updatable source) or overwrite history (insert_overwrite when you needed merge). This skill picks the strategy by **grain × scale × platform** and configures it correctly.

## Inputs

- The model's grain (one row per X per Y).
- Source table(s) and how rows arrive: **immutable append**, **late-arriving**, **upsertable** (status changes, corrections), **fully recomputable**.
- Target platform: dbt-on-Databricks, dbt-on-Snowflake, dbt-on-BigQuery, or hand-rolled SQL.
- Expected daily row volume.

## Outputs

- A model file (or modification) with the right incremental strategy configured.
- Documentation of the strategy choice and trade-offs.

## Procedure

1. **Decide the strategy** using the decision tree below:
   - **Source is immutable append (events, orders-as-placed, log lines)** → `append`. Simplest. Only safe if rows never update.
   - **Source is upsertable (status changes, corrections, SCD2 attribute changes)** → `merge` (Databricks/Snowflake) or `delete+insert` (BigQuery/Redshift). Configure `unique_key`. Slowest of the three.
   - **Partition-aligned full rewrite (rebuild full date partitions)** → `insert_overwrite`. Fastest at scale, idempotent per partition. Use when partitions are independent and can be rebuilt cleanly.
   - **High-volume event table with late-arrivals (≤ watermark)** → `microbatch` (dbt 1.9+) for platforms that support it; otherwise `insert_overwrite` over a trailing window.
2. **Configure the strategy:**
   - `unique_key`: the actual grain key. For composite grain, a list (`[customer_id, valid_from]`).
   - `cluster_by` / `partition_by`: usually the partition column (`event_date`, `order_date`).
   - `on_schema_change`: typically `'append_new_columns'` or `'sync_all_columns'`. Avoid `'fail'` for actively-evolving sources.
   - `event_time`: required for `microbatch`; otherwise optional but useful for `insert_overwrite` window selection.
3. **Use `is_incremental()`** to filter the source on incremental runs:
   - Include a small **lookback window** to catch late-arrivers (e.g., source has 6h watermark → lookback 1 day to be safe).
   - The lookback creates duplicate-candidate rows; rely on `unique_key` (merge) or partition replacement (insert_overwrite) to handle.
4. **Idempotency check:** can you run the same incremental run twice in a row and get the same target state? If no, you have a bug.
5. **Backfill plan.** Document how to rebuild from scratch (`--full-refresh` or equivalent) and what it costs.
6. **Test:** at minimum a uniqueness test on `unique_key`, a not-null test on partition column, and a row-count delta sanity check.

## Big-data / safety guards

- **Don't use `append` on an updatable source.** It will silently skip status changes. Test by mutating a source row and verifying the target updates.
- **`merge` cost grows with target size**, not just incremental delta. For tables > 10B rows, switch to `insert_overwrite` partition-replace.
- **Lookback window must exceed the source's watermark.** Source watermark = 6h; lookback = at least 1 day to safely catch all late-arrivers.
- **Don't drop the production target to test.** Use `dbt build --select <model>+ --vars '{run_full_refresh: true}'` against a dev schema.

## Worked example

> User: "Make `int_sessions_stitched` (one row per `session_id`) incremental on Databricks. Sessions can update for up to 6 hours after start as users keep clicking, then are immutable."

Strategy choice: sessions are upsertable during a 6h window, then immutable. **`merge`** with a 1-day lookback is correct. Don't use `append` (would miss within-session updates), don't use `insert_overwrite` partition-replace (sessions span midnight).

```sql
{{ config(
    materialized = 'incremental',
    incremental_strategy = 'merge',
    unique_key = 'session_id',
    cluster_by = ['session_date'],
    on_schema_change = 'append_new_columns'
) }}

WITH events AS (
  SELECT
      session_id
    , customer_id
    , anonymous_id
    , surface
    , region
    , MIN(event_ts) AS session_start_ts
    , MAX(event_ts) AS session_end_ts
    , CAST(MIN(event_ts) AS DATE) AS session_date
    , COUNT(*) AS event_count
    , MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS purchased
  FROM {{ ref('fct_page_events') }}
  WHERE is_synthetic = FALSE
    {% if is_incremental() %}
      -- Look back 1 day past the source watermark (6h) to catch late-arriving events.
      AND event_date >= (SELECT date_sub(MAX(session_date), 1) FROM {{ this }})
    {% else %}
      AND event_date >= date_sub(current_date(), 90)  -- initial build: 90 days
    {% endif %}
  GROUP BY session_id, customer_id, anonymous_id, surface, region
)
SELECT * FROM events;
```

Tests (in the schema yml):

```yaml
models:
  - name: int_sessions_stitched
    columns:
      - name: session_id
        tests: [unique, not_null]
      - name: session_date
        tests: [not_null]
```

Idempotency: re-running the same hour produces no change (merge on `session_id` is a no-op when the source rows are the same). Late-arriving event lands in the next run because the lookback window includes its `event_date`.

Full-refresh cost: ~120 GB scan over 90 days; target rebuild ~ 8 minutes on the team's `prod` warehouse.

## What to add to docs/ if missing

- A new model added with non-obvious strategy choice → log the rationale at the top of the model file in a comment, referencing this skill.
- A late-arrival pattern that broke an existing model → update the lookback window everywhere it applies; add a note to `docs/pipelines.md` about the source's watermark.
