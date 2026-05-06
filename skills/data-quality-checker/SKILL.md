---
name: data-quality-checker
description: Consumer-side ad-hoc data audit — duplicates, nulls, schema drift, freshness, row-count sanity — before trusting a table for an analysis.
when_to_invoke:
  - "Is this table healthy?"
  - "Validate the data before I use it"
  - "Are there duplicates / nulls / drift in X?"
  - Always before a high-stakes analysis (exec deliverable, model training).
---

# Data Quality Checker

Consumer-side audits. **Different from the producing team's tests** — those are structural; these are "is the data sane *for my use case*." Run before any analysis where being wrong is expensive.

## Inputs

- Full table path.
- Optional: time window (default: last 7 days for `fct_*`, all rows for `dim_*` ≤ 100M).
- Optional: expected uniqueness keys (default: read from `docs/schema/<table>.md`).

## Outputs

- A pass/fail checklist with evidence (row counts, deltas, percentiles).
- Severity per failure: **Block** (don't trust the data), **Warn** (use with caveats), **Info** (note in writeup).

## Procedure

1. **Read `docs/schema/<table>.md`** to know expected partition column, grain key, expected nulls, documented gotchas.
2. **Freshness check.** `MAX(<partition_column>)` — does it match the SLA in `docs/pipelines.md`? **Block** if data is older than SLA + 2 hours.
3. **Row count trend.** Last 14 days of daily row counts. Flag anything outside ±3σ of trailing-7-day mean (excluding the last day itself). **Warn** on a 3σ outlier; **Block** on a 5σ outlier.
4. **Duplicate check** on the documented grain key:

```sql
SELECT <grain_key>, COUNT(*) AS dupes
FROM <full_path>
WHERE <partition_column> >= ...
GROUP BY <grain_key>
HAVING COUNT(*) > 1;
```

If the result is non-empty: **Block** unless the duplicates are documented (e.g., the `fct_page_events` Monday-duplicate gotcha).

5. **Null-rate drift.** For each column, compare current null rate to the trailing-7-day mean. Flag any column with > 10pt absolute change.
6. **Schema drift.** Compare current `DESCRIBE` output to the column list in `docs/schema/<table>.md`. Flag added, removed, or type-changed columns.
7. **Cross-table referential integrity** (when relevant). E.g., `fct_orders.customer_id` should always exist in `dim_customers`. Spot-check on a sample.
8. **Cross-source reconciliation** (high-stakes only). E.g., `COUNT(*) FROM fct_orders WHERE status = 'completed'` should approximately equal the finance ledger's order count for the same window. Tolerance per the team — typically < 1% delta. **Block** on a > 5% gap.
9. **Output a pass/fail block** with severities and evidence. Don't soften with hedge words — be explicit.

## Big-data / safety guards

- All checks must hit partitioned tables with the partition filter.
- For tables > 1B rows, sample row-level checks (5% with `random_state=SEED`).
- **Reconcile against finance only on Internal/Confidential channels** — these counts can imply revenue magnitude.

## Worked example

> User: "I'm about to compute monthly GMV for the exec deck. Validate `fct_orders` first."

1. Read `docs/schema/fct_orders.md`. Partition: `order_date`. Grain: `order_id`. SLA: 03:00 UTC.
2. **Freshness:** `MAX(order_date) = current_date() - 1`, `MAX(updated_at) = today 02:47 UTC`. Within SLA. **Pass.**
3. **Row count trend:** last 14 days, current is +0.3σ from trailing mean. **Pass.**
4. **Duplicates** on `order_id` for last 30 days: 0. **Pass.**
5. **Null-rate drift:**
   - `cost_of_goods_sold` null rate stable at 0.3% (matches docs).
   - `discount_codes` null rate jumped from 12% → 24% on `2025-01-12`. **Warn** — investigate before using `discount_codes` in any analysis. Likely upstream issue.
6. **Schema:** matches `docs/schema/fct_orders.md`. **Pass.**
7. **Referential integrity:** sampled 1000 `customer_id`s; all join successfully to `dim_customers`. **Pass.**
8. **Reconciliation with finance:** completed-order count for Dec 2024 = 18.4M; finance ledger reports 18.5M (delta 0.5%). Within tolerance. **Pass.**

```
DQ check — fct_orders for Dec 2024:
  ✓ Freshness (within SLA)
  ✓ Row counts (within ±3σ)
  ✓ No duplicate order_ids
  ⚠ discount_codes null rate stepped up on 2025-01-12 (12% → 24%)
  ✓ Schema matches docs
  ✓ Referential integrity sample passed
  ✓ Finance reconciliation 0.5% delta (within tolerance)

Verdict: SAFE for monthly GMV (does not depend on discount_codes).
Open warning: investigate discount_codes nullification before any promo analysis.
```

## What to add to docs/ if missing

- A new gotcha (e.g., `discount_codes` null spike) → add to `docs/schema/fct_orders.md` Gotchas.
- A new structural test (e.g., "always reconcile completed orders with finance") → add to `docs/pipelines.md`'s "Failure runbook" or a team-level "before exec deliverables" checklist.
