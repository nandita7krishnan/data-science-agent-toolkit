# `analytics_prod.core.fct_page_events`

## Overview

- **Full path:** `analytics_prod.core.fct_page_events`
- **Owner:** Data Eng — Clickstream (Slack: `#data-eng-clickstream`)
- **Grain:** **One row per tracked event.** Web and app event streams are unioned here.
- **Type:** Fact (event stream, append-only, partitioned)
- **Last reviewed:** 2025-01-15

> **HIGH-VOLUME WARNING.** This table is ~3 billion rows / day. A query against it without a partition filter will scan tens of TB and **be killed by the warehouse**, but only after running for many minutes. **Always filter on `event_date` first.**

## Partitioning & filters

- **Partition column:** `event_date` (`DATE`)
- **Required filter:** `WHERE event_date >= ...`. **No exceptions.** Full scan is **~120 TB**.
- **Recommended date range:** A single day for ad-hoc EDA. ≤ 7 days for funnel work. ≤ 30 days for cohort work; if you need longer, sample.
- **Sub-partition:** Internally partitioned by `event_date`, then by `surface` (web/ios/android). Filtering on `surface` is also a meaningful cost reduction.

## Freshness

- **Update frequency:** Hourly (top of hour + 90 min for the prior hour's partition).
- **Typical lag (event-time → table):** ~90 min.
- **Watermark:** 6h. Events that arrive > 6h late are routed to `fct_page_events_late` and **do not** backfill the original `event_date` partition.
- **How to check:** `SELECT MAX(event_ts) FROM analytics_prod.core.fct_page_events WHERE event_date = current_date();`
- **Producing job:** `fct_page_events_load` (see `docs/pipelines.md`).

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `event_id` | `STRING` | N | Internal | Primary key. Format: `evt_<ulid>`. |
| `event_date` | `DATE` | N | Internal | **Partition column.** UTC date of `event_ts`. |
| `event_ts` | `TIMESTAMP` | N | Internal | Event time in UTC. |
| `event_name` | `STRING` | N | Internal | Canonical event name. See "Allowed values" below. |
| `customer_id` | `STRING` | Y | PII | NULL for unidentified visitors. Joins to `dim_customers.customer_id`. |
| `anonymous_id` | `STRING` | N | Internal | Always present. Pseudonymous device-level ID. **Quasi-PII** when joined to direct identifiers. |
| `session_id` | `STRING` | N | Internal | A 30-min-inactivity-timeout session window. Same `anonymous_id` can have many sessions. |
| `surface` | `STRING` | N | Internal | `'web' | 'ios' | 'android'`. |
| `region` | `STRING` | N | Internal | `'us' | 'ca' | 'uk' | 'au'`. |
| `page_url` | `STRING` | Y | Internal | Web only. NULL on app surfaces. |
| `page_path` | `STRING` | Y | Internal | Web only. Path without query string or fragment. |
| `screen_name` | `STRING` | Y | Internal | App only. NULL on web. |
| `referrer_url` | `STRING` | Y | Internal | Web only. NULL on app. |
| `utm_source` | `STRING` | Y | Internal | Marketing UTM. NULL when no campaign tagging. |
| `utm_medium` | `STRING` | Y | Internal | Marketing UTM. |
| `utm_campaign` | `STRING` | Y | Internal | Marketing UTM. |
| `device_id` | `STRING` | Y | PII | App device identifier. NULL on web. |
| `user_agent` | `STRING` | Y | Internal | Web only. Use for bot filtering. |
| `ip_address` | `STRING` | Y | PII | Truncated to /24 for IPv4 at write time (still quasi-PII). |
| `country_code` | `STRING` | N | Internal | ISO-3166 alpha-2 from IP geolocation. |
| `properties` | `MAP<STRING, STRING>` | Y | Internal | Per-event payload (e.g., `product_id`, `cart_value`). Schema varies by `event_name`. |
| `is_synthetic` | `BOOLEAN` | N | Internal | TRUE for load-test traffic. **Always exclude in analyses.** |

### Allowed values for `event_name`

Canonical analytics events. Other values exist but are unstable; ignore unless you have a specific reason.

- `page_view` — any page or screen impression.
- `product_view` — product detail page (PDP) impression.
- `add_to_cart` — item added to cart. `properties` includes `product_id`, `sku_id`, `quantity`, `unit_price`.
- `remove_from_cart` — item removed.
- `begin_checkout` — checkout flow started.
- `add_payment_info` — payment method entered.
- `purchase` — order completed. `properties` includes `order_id`. **Cross-reference against `fct_orders`** if it matters — clickstream `purchase` and warehouse-of-record `fct_orders.status = 'completed'` can disagree by ~0.5% (events lost, attribution timing).

### `properties` payloads (common)

| Event | Required keys |
|---|---|
| `product_view` | `product_id`, `sku_id` |
| `add_to_cart` | `product_id`, `sku_id`, `quantity`, `unit_price` |
| `purchase` | `order_id`, `order_subtotal`, `currency` |
| `page_view` | none required |

## Relationships

- `customer_id` → `dim_customers.customer_id` (with `is_current = TRUE`). Many `customer_id` rows are NULL — that's expected.
- `session_id` → `fct_orders.session_id` (1 session : 0..1 orders).
- `properties['product_id']` → `dim_products.product_id` (when present).

## Gotchas

- **The "Monday duplicate" gotcha.** The Kafka producer retries on weekends when a downstream topic is paused for maintenance. **Mondays before 06:00 UTC have ~3% duplicate event rows.** Deduplicate on `event_id` if precision matters:

  ```sql
  WITH dedup AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY event_ts) AS rn
    FROM analytics_prod.core.fct_page_events
    WHERE event_date = DATE '2025-01-13'  -- a Monday
  )
  SELECT * FROM dedup WHERE rn = 1;
  ```

- **`customer_id IS NULL` is normal.** ~40% of `page_view` events are from unidentified visitors. For DAU-style metrics, the canonical formula in `docs/metric-definitions.md` filters to `customer_id IS NOT NULL` *and* requires a qualifying event — match it exactly.
- **Synthetic load tests.** A nightly load test produces ~50M synthetic events. **Always filter `is_synthetic = FALSE`** unless you specifically want to study the load test.
- **Bot traffic** is partially scrubbed in the source (UA-based filter), but ~1% leaks through. For high-fidelity work add `user_agent NOT ILIKE '%bot%' AND user_agent NOT ILIKE '%crawler%' AND user_agent NOT ILIKE '%spider%'`.
- **App `purchase` events are sometimes missing** when the user closes the app immediately after the order goes through. **Trust `fct_orders` for revenue, trust `fct_page_events.purchase` only for funnel timing.**
- **Sessions cross midnight.** A session that starts at 23:55 UTC and ends at 00:10 UTC produces events under two `event_date` partitions but one `session_id`. If you're sessionizing, query both days and dedupe on `session_id`.
- **Anonymous → identified handoff.** When a visitor logs in mid-session, only events *after* login have `customer_id` set. Events before login still have `anonymous_id`. Stitching is done downstream in `int_sessions_stitched` (not documented in this repo).

## Example queries

### Sanity check — last hour of events

```sql
SELECT
    surface
  , COUNT(*) AS events_last_hour
FROM analytics_prod.core.fct_page_events
WHERE event_date = current_date()
  AND event_ts >= current_timestamp() - INTERVAL 1 HOUR
  AND is_synthetic = FALSE
GROUP BY surface;
```

### DAU — single day, exact formula

```sql
SELECT COUNT(DISTINCT customer_id) AS dau
FROM analytics_prod.core.fct_page_events
WHERE event_date = DATE '2025-01-15'
  AND event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
  AND is_synthetic = FALSE
  AND customer_id IS NOT NULL;
```

### PDP-to-purchase funnel — single day, single surface

```sql
WITH events AS (
  SELECT session_id, event_name
  FROM analytics_prod.core.fct_page_events
  WHERE event_date = DATE '2025-01-15'
    AND surface = 'web'
    AND is_synthetic = FALSE
    AND event_name IN ('product_view','add_to_cart','begin_checkout','purchase')
),
session_steps AS (
  SELECT
      session_id
    , MAX(CASE WHEN event_name = 'product_view'  THEN 1 ELSE 0 END) AS step_pdp
    , MAX(CASE WHEN event_name = 'add_to_cart'   THEN 1 ELSE 0 END) AS step_atc
    , MAX(CASE WHEN event_name = 'begin_checkout' THEN 1 ELSE 0 END) AS step_chk
    , MAX(CASE WHEN event_name = 'purchase'      THEN 1 ELSE 0 END) AS step_buy
  FROM events
  GROUP BY session_id
)
SELECT
    SUM(step_pdp) AS pdp_sessions
  , SUM(step_atc) AS atc_sessions
  , SUM(step_chk) AS checkout_sessions
  , SUM(step_buy) AS purchase_sessions
FROM session_steps;
```

## Change log

- `2025-01-15` (DS): Documented Monday-duplicate gotcha after a metric debug session.
- `2024-08-22` (DE): IP truncation to /24 enabled at write time for GDPR alignment.
