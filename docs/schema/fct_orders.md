# `analytics_prod.core.fct_orders`

## Overview

- **Full path:** `analytics_prod.core.fct_orders`
- **Owner:** Data Eng — Commerce (Slack: `#data-eng-commerce`)
- **Grain:** **One row per order.** A "cart" with multiple items is one row; line items live in `fct_order_items` (joined on `order_id`).
- **Type:** Fact (transactional, partitioned)
- **Last reviewed:** 2025-01-15

## Partitioning & filters

- **Partition column:** `order_date` (`DATE`)
- **Required filter:** `WHERE order_date >= ...`. Full scan is **~2.4 TB** — don't do it. The query will be killed by the warehouse.
- **Recommended date range:** Last 90 days for most analyses; trailing 12 months for cohort/LTV work.
- **Clustering:** None (low-cardinality clustering not worth it at this volume).

## Freshness

- **Update frequency:** Daily at 01:30 UTC.
- **Typical lag (event-time → table):** ~3 hours from order placement.
- **Watermark:** 24h — orders that land in the source replica more than 24h after `created_at` go to `fct_orders_late` (rare; usually source-system retries).
- **How to check:** `SELECT MAX(order_date) FROM analytics_prod.core.fct_orders;`
- **Producing job:** `fct_orders_load` (see `docs/pipelines.md`).

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `order_id` | `STRING` | N | Internal | Primary key. Format: `ord_<26 chars>` (ULID). |
| `order_date` | `DATE` | N | Internal | **Partition column.** Date the order was placed (UTC). |
| `order_ts` | `TIMESTAMP` | N | Internal | Exact placement timestamp (UTC). |
| `customer_id` | `STRING` | N | PII | Identified customer. Joins to `dim_customers.customer_id`. |
| `session_id` | `STRING` | Y | Internal | Session in `fct_page_events` that produced the order. NULL for orders placed via channels without web/app sessions (rare). |
| `surface` | `STRING` | N | Internal | `'web' | 'ios' | 'android'`. |
| `region` | `STRING` | N | Internal | `'us' | 'ca' | 'uk' | 'au'`. |
| `currency` | `STRING` | N | Internal | ISO 4217. `'USD' | 'CAD' | 'GBP' | 'AUD'`. **Always check region+currency before summing across rows.** |
| `status` | `STRING` | N | Internal | `'pending' | 'completed' | 'cancelled' | 'refunded' | 'partially_refunded' | 'chargeback'`. See `docs/business-info.md`. |
| `order_subtotal` | `DECIMAL(12,2)` | N | Internal | Pre-tax, pre-shipping merchandise total. Sum this for **GMV**. |
| `order_total` | `DECIMAL(12,2)` | N | Internal | Final amount charged to the customer (subtotal + tax + shipping − discounts). |
| `tax_amount` | `DECIMAL(12,2)` | N | Internal | Tax (always ≥ 0). |
| `shipping_amount` | `DECIMAL(12,2)` | N | Internal | Shipping (always ≥ 0). |
| `discount_amount` | `DECIMAL(12,2)` | N | Internal | Total discounts applied (positive number). |
| `discount_codes` | `ARRAY<STRING>` | Y | Internal | Promotion codes used. Empty array `[]` for no codes. |
| `payment_method` | `STRING` | N | Internal | `'card' | 'paypal' | 'apple_pay' | 'google_pay' | 'gift_card' | 'store_credit'`. |
| `is_first_order` | `BOOLEAN` | N | Internal | Lifetime first completed order for this `customer_id`. Computed at write time, immutable. |
| `cost_of_goods_sold` | `DECIMAL(12,2)` | Y | **Confidential** | COGS at time of order. NULL for orders before 2024-03-01 (not backfilled). |
| `gross_margin` | `DECIMAL(12,2)` | Y | **Confidential** | `order_subtotal - cost_of_goods_sold`. NULL when COGS NULL. |
| `created_at` | `TIMESTAMP` | N | Internal | Source-system insert time. Use `order_ts` for analytics. |
| `updated_at` | `TIMESTAMP` | N | Internal | Last status change in the source. Useful for "orders updated today" queries. |

### Allowed values quick reference

- `status`: `pending | completed | cancelled | refunded | partially_refunded | chargeback`
- `surface`: `web | ios | android`
- `region`: `us | ca | uk | au`
- `payment_method`: `card | paypal | apple_pay | google_pay | gift_card | store_credit`

## Relationships

- `order_id` → `fct_order_items.order_id` (1:N — line items per order)
- `order_id` → `fct_refunds.order_id` (1:N — multiple refunds possible per order)
- `customer_id` → `dim_customers.customer_id` **with `dim_customers.is_current = TRUE`** for current attributes; without that filter you get all SCD2 versions and a row explosion.
- `session_id` → `fct_page_events.session_id` (1:N — many events per session, one or zero orders).

## Gotchas

- **`status = 'completed'` is not the only "real" order.** Refunded and partially-refunded orders *did* happen — they should be in GMV, then subtracted in net-revenue calculations. Excluding them entirely understates revenue.
- **Cancellations.** `status = 'cancelled'` rows have `order_total > 0` (the original amount) but were never charged. **Always filter them out for revenue numbers.**
- **`order_total` ≠ `order_subtotal`.** GMV uses `order_subtotal`. AOV uses `order_subtotal`. The customer's bank statement uses `order_total`. Don't mix them.
- **Currency.** `SUM(order_subtotal)` across regions without converting is **wrong**. Either filter to one region or convert to a base currency using a daily FX table (not in this repo — ask Finance Data).
- **`is_first_order` is computed at write time.** A customer who placed order #2 yesterday will have `is_first_order = FALSE` on it, but if order #1 was refunded after the fact, `is_first_order` is *not* recomputed. For "first paid order ever," recompute from a `MIN(order_date)` window. For "first order placed (regardless of outcome)," trust the column.
- **`cost_of_goods_sold` is Confidential.** Don't paste this column into Slack, screenshots, or external decks. Aggregated margin numbers are fine for internal stakeholders.
- **Timezone.** All dates/timestamps are UTC. A US-East customer placing an order at 11pm local will land on the *next* `order_date` in UTC. For local-time DAUs use `from_utc_timestamp()` and bucket on the local date; for company-wide reporting, leave as UTC.

## Example queries

### Sanity check — is today's data here?

```sql
SELECT
    MAX(order_date) AS latest_order_date
  , COUNT(*) AS rows_today
FROM analytics_prod.core.fct_orders
WHERE order_date = current_date();
```

### Daily completed orders + GMV — last 30 days

```sql
SELECT
    order_date
  , region
  , COUNT(*) AS completed_orders
  , SUM(order_subtotal) AS gmv
FROM analytics_prod.core.fct_orders
WHERE order_date >= date_sub(current_date(), 30)
  AND status = 'completed'
GROUP BY order_date, region
ORDER BY order_date, region;
```

### First-order cohort with current customer attributes

```sql
SELECT
    o.order_date AS first_order_date
  , c.first_touch_channel
  , COUNT(DISTINCT o.customer_id) AS new_buyers
  , SUM(o.order_subtotal) AS first_order_gmv
FROM analytics_prod.core.fct_orders o
JOIN analytics_prod.core.dim_customers c
  ON c.customer_id = o.customer_id
 AND c.is_current = TRUE
WHERE o.order_date >= date_sub(current_date(), 90)
  AND o.status = 'completed'
  AND o.is_first_order = TRUE
GROUP BY o.order_date, c.first_touch_channel
ORDER BY o.order_date, new_buyers DESC;
```

## Change log

- `2025-01-15` (DS): Documented `cost_of_goods_sold` as Confidential after a near-miss in a Slack screenshot.
- `2024-11-04` (DE): Added `is_first_order` (write-time-computed). Backfilled to 2022-01-01.
