---
name: eda-profiler
description: Profile any table — row counts, nulls, distributions, ranges, anomalies — sample-aware for big data, with output as a structured markdown summary.
when_to_invoke:
  - "Profile this table"
  - "Run EDA on X"
  - "What does X look like?"
  - "Are there any anomalies in X?"
---

# EDA Profiler

Produce a one-page, structured profile of a table. Designed to be the **first** thing the agent runs against a new table, before any analytical query.

## Inputs

- Full table path (e.g., `analytics_prod.core.fct_orders`).
- Optional: date range (default: most recent 1 day for `fct_*`, all rows for `dim_*` < 100M rows).
- Optional: focus columns (default: profile all columns).

## Outputs

A markdown summary with five sections:

1. **Overview** — row count, distinct count of the grain key, partition coverage.
2. **Schema** — columns, types, null rate.
3. **Numerics** — min, max, mean, median, p25/p75/p95/p99, stddev for each numeric column.
4. **Categoricals** — top 10 values per categorical column with counts.
5. **Anomalies** — flagged columns: >50% null, single-value, unexpected types, out-of-range values.

## Procedure

1. **Read `docs/schema/<table>.md`** if it exists. The doc tells you the partition column, expected nulls, and known gotchas — anchor the profile against it.
2. **Apply the partition filter.** No exceptions. For `fct_*` tables, default to a single recent day.
3. **Sample if needed.** If the filtered slice is > 10M rows, sample to 1% (`TABLESAMPLE (1 PERCENT)` or `df.sample(fraction=0.01, seed=SEED)`). State the sample size and seed in the output.
4. **Compute overview** — `COUNT(*)`, `COUNT(DISTINCT <grain_key>)`, `MIN(partition_col)`, `MAX(partition_col)`.
5. **Per-column stats** in one query each (or one batched query). For numerics: `MIN`, `MAX`, `AVG`, `STDDEV`, `approx_percentile(col, array(0.25, 0.5, 0.75, 0.95, 0.99))`. For categoricals: `GROUP BY col ORDER BY COUNT(*) DESC LIMIT 10`. For dates/timestamps: `MIN`, `MAX`, count by date for trend.
6. **Flag anomalies.** Null rate > 50%, single-value column (cardinality = 1), value outside the documented range, type mismatch with `docs/schema/<table>.md`.
7. **Write the summary** as markdown. Compare against `docs/schema/<table>.md` if it exists; call out any drift.

## Big-data / safety guards

- **Never** run on a partitioned table without filtering on the partition column.
- For tables > 10M rows in the filtered slice, **always sample** and state the sample size + seed.
- For PII columns (per `docs/data-classification.md`), do **not** show `top 10 values` — show count, null rate, and cardinality only. Never paste raw PII into an EDA summary.

## Worked example

> User: "Profile `fct_orders` for the last day."

1. Read `docs/schema/fct_orders.md`. Partition is `order_date`; no `is_test_account` to exclude.
2. Filter: `WHERE order_date = current_date()` — about 600K rows, well under sampling threshold.
3. Run profiling SQL (one batched query when possible):

```sql
SELECT
    COUNT(*) AS row_count
  , COUNT(DISTINCT order_id) AS distinct_orders
  , COUNT(DISTINCT customer_id) AS distinct_customers
  , MIN(order_date) AS min_date, MAX(order_date) AS max_date
  , AVG(order_subtotal) AS avg_subtotal
  , approx_percentile(order_subtotal, array(0.5, 0.95, 0.99)) AS subtotal_p50_p95_p99
  , SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) / COUNT(*) AS completed_share
FROM analytics_prod.core.fct_orders
WHERE order_date = current_date();
```

4. Final summary:

```
fct_orders — order_date = 2025-01-15
- Rows: 612,403 (1:1 with order_id, no duplicates)
- Distinct customers: 587,114
- Subtotal: avg $84.12, p50 $52.00, p95 $267.99, p99 $612.40
- Status mix: completed 91.2%, pending 4.1%, cancelled 3.4%, refunded/other 1.3%
- Anomalies: cost_of_goods_sold null on 0.3% of rows (matches docs — pre-2024-03-01 backfill)
- All other columns within expected ranges per docs/schema/fct_orders.md
```

## What to add to docs/ if missing

- A schema doc for the table → run `schema-documenter` skill.
- A new gotcha discovered during profiling (e.g., unexpected null spike) → propose addition to the table's `docs/schema/<table>.md` Gotchas section.
