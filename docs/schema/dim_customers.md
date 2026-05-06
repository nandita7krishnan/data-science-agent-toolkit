# `analytics_prod.core.dim_customers`

## Overview

- **Full path:** `analytics_prod.core.dim_customers`
- **Owner:** Data Eng — Customer (Slack: `#data-eng-customer`)
- **Grain:** **One row per `customer_id` per validity window.** A customer with three changes to their tracked attributes has four rows (one for each time-window). At any given point in time, exactly one row has `is_current = TRUE`.
- **Type:** Dimension, **Slowly Changing Dimension Type 2 (SCD2)**.
- **Last reviewed:** 2025-01-15

> **The `is_current = TRUE` filter is the most important rule in this doc.** If you don't include it, you get every historical version of every customer and your row counts will be wrong by 2-5×.

## Partitioning & filters

- **Partition column:** None. Table is small (~50M current customers, ~110M total rows including history).
- **Required filter for current state:** `WHERE is_current = TRUE`. Use this **always**, unless you're explicitly asking a point-in-time historical question.
- **Required filter for as-of-date queries:** `WHERE valid_from <= as_of_date AND (valid_to > as_of_date OR valid_to IS NULL)`.

## Freshness

- **Update frequency:** Daily at 02:00 UTC.
- **Typical lag:** Same-day. New customer signups appear by the next 02:00 run.
- **How to check:** `SELECT MAX(updated_at) FROM analytics_prod.core.dim_customers WHERE is_current = TRUE;`
- **Producing job:** `dim_customers_scd2_refresh` (see `docs/pipelines.md`).

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `customer_id` | `STRING` | N | PII | Primary key (per validity window). Stable across versions. |
| `valid_from` | `TIMESTAMP` | N | Internal | This row became active at this UTC timestamp. |
| `valid_to` | `TIMESTAMP` | Y | Internal | This row was superseded at this UTC timestamp. **NULL means still current.** |
| `is_current` | `BOOLEAN` | N | Internal | TRUE iff `valid_to IS NULL` (denormalized for filter convenience). |
| `email` | `STRING` | N | **PII** | Lowercased. Quasi-key (an email change creates a new validity window). |
| `first_name` | `STRING` | Y | **PII** |  |
| `last_name` | `STRING` | Y | **PII** |  |
| `signup_ts` | `TIMESTAMP` | N | Internal | Account creation. **Immutable** — same across all validity windows. |
| `signup_surface` | `STRING` | N | Internal | `'web' | 'ios' | 'android'`. Immutable. |
| `signup_region` | `STRING` | N | Internal | `'us' | 'ca' | 'uk' | 'au'`. Immutable. |
| `region` | `STRING` | N | Internal | Current region. Set at first order; immutable thereafter. (May differ from `signup_region` if customer signed up in one region but first ordered in another.) |
| `first_touch_channel` | `STRING` | Y | Internal | Marketing channel of the first session that led to signup. Immutable. NULL for organic/unknown attribution. |
| `first_order_date` | `DATE` | Y | Internal | Date of first completed order. NULL for prospects (signed up but never bought). |
| `lifecycle_stage` | `STRING` | N | Internal | `'prospect' | 'first_time_buyer' | 'repeat_buyer' | 'vip' | 'at_risk' | 'churned' | 'reactivated'`. Recomputed daily. |
| `lifetime_orders` | `BIGINT` | N | Internal | Count of completed orders to date. Recomputed daily. |
| `lifetime_value` | `DECIMAL(14,2)` | Y | **Confidential** | Sum of net revenue. NULL when no completed orders. **Confidential** when joined to direct identifiers. |
| `is_employee` | `BOOLEAN` | N | Internal | TRUE if ABC Business staff. Use to filter out internal traffic for external metrics. |
| `is_test_account` | `BOOLEAN` | N | Internal | TRUE for QA accounts. **Always exclude in production analyses.** |
| `marketing_consent` | `BOOLEAN` | N | Internal | Whether the customer has consented to marketing email. Tracked for GDPR/CCPA. Changes here trigger a new validity window. |
| `created_at` | `TIMESTAMP` | N | Internal | Source row insert time. |
| `updated_at` | `TIMESTAMP` | N | Internal | Source row last-update time. Used by the SCD2 job for diffing. |

### What triggers a new validity window?

A new SCD2 row is created when **any of these tracked attributes** change:

- `email`, `first_name`, `last_name`
- `region`, `first_touch_channel`
- `marketing_consent`

Changes to **untracked** columns (`lifecycle_stage`, `lifetime_orders`, `lifetime_value`) update the **current** row in place — they don't create a new validity window.

### Allowed values

- `lifecycle_stage`: `prospect | first_time_buyer | repeat_buyer | vip | at_risk | churned | reactivated`
- `region`, `signup_region`: `us | ca | uk | au`
- `signup_surface`: `web | ios | android`

## Relationships

- `customer_id` → `fct_orders.customer_id` (1:N — multiple orders per customer)
- `customer_id` → `fct_page_events.customer_id` (1:N — many events per customer; many events without `customer_id`)
- `customer_id` → `dim_customer_addresses.customer_id` (1:N — multiple addresses)

## Gotchas

- **The "no `is_current` filter" gotcha.** This is the #1 source of wrong numbers in this repo. A customer with 3 prior name changes contributes 4 rows. Joins to `fct_orders` without the filter explode rows by the average history depth. **Always include `WHERE c.is_current = TRUE`** unless you're doing a point-in-time analysis.
- **`first_order_date` lags.** It's set by `dim_customers_lifecycle_refresh` at 04:00 UTC, after `fct_orders_load`. New first-time buyers from yesterday appear with `first_order_date = NULL` until the lifecycle job runs.
- **Test accounts are real customer_ids in the source.** They have realistic emails and don't get filtered out unless you do it. **`AND is_test_account = FALSE AND is_employee = FALSE`** for any external metric.
- **`region` vs `signup_region`.** If a customer signed up in `us` but their first order was in `uk` (uncommon, but happens with VPN / travel), `region = 'uk'` and `signup_region = 'us'`. Pick deliberately based on what the analysis is asking.
- **`lifetime_value` is Confidential.** Join carefully and don't paste the column externally with names attached. See `docs/data-classification.md`.
- **Marketing consent revocation.** When `marketing_consent` flips from TRUE→FALSE, downstream systems are *eventually* consistent. The data warehouse is updated within 24h; do not rely on `dim_customers` for *send-time* consent — use the marketing platform's API.
- **Email is lowercased here**, but **case-sensitive in some source systems.** If you're matching against an external email list, lowercase both sides.

## Example queries

### Sanity check — current customer count

```sql
SELECT
    COUNT(*) AS current_customers
  , MAX(updated_at) AS latest_update
FROM analytics_prod.core.dim_customers
WHERE is_current = TRUE
  AND is_test_account = FALSE
  AND is_employee = FALSE;
```

### Joining to orders with current customer attributes

```sql
SELECT
    c.first_touch_channel
  , c.region
  , COUNT(DISTINCT o.customer_id) AS buyers
  , SUM(o.order_subtotal) AS gmv
FROM analytics_prod.core.fct_orders o
JOIN analytics_prod.core.dim_customers c
  ON c.customer_id = o.customer_id
 AND c.is_current = TRUE
WHERE o.order_date >= date_sub(current_date(), 30)
  AND o.status = 'completed'
  AND c.is_test_account = FALSE
  AND c.is_employee = FALSE
GROUP BY c.first_touch_channel, c.region
ORDER BY gmv DESC;
```

### Point-in-time: who was a "vip" on 2024-12-25?

```sql
-- Use valid_from / valid_to, not is_current.
-- Note: lifecycle_stage updates in place on the current row, so historical lifecycle is not preserved
-- in dim_customers. For accurate point-in-time lifecycle, use the snapshot table dim_customers_snapshot
-- (one snapshot per day) — not in scope for this repo.
SELECT customer_id, email
FROM analytics_prod.core.dim_customers
WHERE TIMESTAMP '2024-12-25 00:00:00' >= valid_from
  AND (valid_to > TIMESTAMP '2024-12-25 00:00:00' OR valid_to IS NULL)
  AND lifecycle_stage = 'vip'        -- !! NOT a true point-in-time read; see comment above
;
```

> **Important:** `lifecycle_stage` updates in place. The point-in-time query above gives the *current* lifecycle on the row valid at 2024-12-25, not the lifecycle the customer had on 2024-12-25. For true point-in-time lifecycle history, query a daily snapshot (out of scope for this repo).

## Change log

- `2025-01-15` (DS): Clarified that `lifecycle_stage` does **not** preserve point-in-time history (gotcha that surfaced in a churn analysis).
- `2024-09-12` (DE): Added `marketing_consent` to tracked-attribute set.
