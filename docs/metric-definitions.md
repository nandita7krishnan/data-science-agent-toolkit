# Metric Definitions

Canonical formulas. **Use these exactly — do not improvise alternatives.** If the metric you need isn't here, propose adding it before computing.

> SQL is Spark SQL (ANSI-leaning). All examples assume `analytics_prod.core` catalog.

---

## DAU — Daily Active Users

- **Definition:** Count of distinct `customer_id`s with at least one qualifying event on the given day.
- **Qualifying event:** `event_name IN ('page_view', 'product_view', 'add_to_cart', 'begin_checkout', 'purchase')`. Excludes synthetic events (`is_synthetic = true`) and internal traffic (see exclusions below).
- **Source:** `analytics_prod.core.fct_page_events`
- **Formula:**

```sql
SELECT
  event_date,
  COUNT(DISTINCT customer_id) AS dau
FROM analytics_prod.core.fct_page_events
WHERE event_date = DATE '2025-01-15'
  AND event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
  AND is_synthetic = FALSE
  AND customer_id IS NOT NULL
  AND NOT (
    user_agent ILIKE '%bot%'
    OR ip_address IN (SELECT ip_address FROM analytics_prod.core.dim_internal_ips)
  )
GROUP BY event_date;
```

- **Exclusions:**
  - `customer_id IS NULL` (anonymous visitors don't count toward DAU; report separately as "anonymous DAU" if asked).
  - Bot user-agents and known internal IP ranges.
  - Synthetic load-test events.
- **Gotchas:**
  - `event_date` is event-time, not server-time. Late-arriving events can move historical DAU by < 0.5% — see `docs/pipelines.md` for the watermark.
  - Do **not** use `fct_orders` for DAU — it under-counts because not all active customers buy on a given day.

---

## WAU / MAU

- **WAU:** Distinct customers with a qualifying event in the trailing **7 days** (rolling, not calendar week).
- **MAU:** Distinct customers with a qualifying event in the trailing **28 days** (not 30 — keeps day-of-week comparable).

```sql
SELECT
  reporting_date,
  COUNT(DISTINCT customer_id) AS wau
FROM analytics_prod.core.fct_page_events
WHERE event_date BETWEEN DATE_SUB(DATE '2025-01-15', 6) AND DATE '2025-01-15'
  AND event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
  AND is_synthetic = FALSE
  AND customer_id IS NOT NULL;
```

- **Gotcha:** WAU/MAU are not the sum of DAU. Don't divide MAU by 28 to "back out" DAU.

---

## 7-day retention (D7)

- **Definition:** Of customers active on day D0, the share who are active again on day D0+7 *exactly* (not "within 7 days").
- **Numerator:** Distinct customers active on `D0` AND active on `D0 + 7`.
- **Denominator:** Distinct customers active on `D0`.
- **Formula:**

```sql
WITH d0 AS (
  SELECT DISTINCT customer_id
  FROM analytics_prod.core.fct_page_events
  WHERE event_date = DATE '2025-01-08'
    AND event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
    AND is_synthetic = FALSE
    AND customer_id IS NOT NULL
),
d7 AS (
  SELECT DISTINCT customer_id
  FROM analytics_prod.core.fct_page_events
  WHERE event_date = DATE '2025-01-15'
    AND event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
    AND is_synthetic = FALSE
    AND customer_id IS NOT NULL
)
SELECT
  COUNT(DISTINCT d7.customer_id) * 1.0 / COUNT(DISTINCT d0.customer_id) AS d7_retention
FROM d0 LEFT JOIN d7 USING (customer_id);
```

- **Gotchas:**
  - "Day 7" means *exactly* the 7th day after D0, not "anytime in the next 7 days." For "anytime within N days" use **rolling retention** below.
  - When charting, plot retention by `D0` cohort date — not by reporting date.

## Rolling N-day retention

- **Definition:** Of customers active on D0, the share active on **any day** in `[D0+1, D0+N]`.

```sql
WITH d0 AS (
  SELECT DISTINCT customer_id, DATE '2025-01-08' AS d0_date
  FROM analytics_prod.core.fct_page_events
  WHERE event_date = DATE '2025-01-08' AND customer_id IS NOT NULL
),
d1_to_d7 AS (
  SELECT DISTINCT customer_id
  FROM analytics_prod.core.fct_page_events
  WHERE event_date BETWEEN DATE '2025-01-09' AND DATE '2025-01-15'
    AND customer_id IS NOT NULL
)
SELECT
  COUNT(DISTINCT d1_to_d7.customer_id) * 1.0 / COUNT(DISTINCT d0.customer_id) AS rolling_d7_retention
FROM d0 LEFT JOIN d1_to_d7 USING (customer_id);
```

---

## Churn rate (180-day)

- **Definition:** Share of customers active in the previous 180-day window who have **no completed orders** in the next 180-day window.
- **Cohort:** All customers with at least one completed order ever (lifetime). Unbounded cohort, not new-buyer cohort.
- **Source:** `analytics_prod.core.fct_orders`, `analytics_prod.core.dim_customers`

```sql
WITH active_prev AS (
  SELECT DISTINCT customer_id
  FROM analytics_prod.core.fct_orders
  WHERE order_date BETWEEN DATE_SUB(DATE '2025-01-15', 360) AND DATE_SUB(DATE '2025-01-15', 181)
    AND status = 'completed'
),
active_curr AS (
  SELECT DISTINCT customer_id
  FROM analytics_prod.core.fct_orders
  WHERE order_date BETWEEN DATE_SUB(DATE '2025-01-15', 180) AND DATE '2025-01-15'
    AND status = 'completed'
)
SELECT
  1.0 - COUNT(DISTINCT active_curr.customer_id) * 1.0 / COUNT(DISTINCT active_prev.customer_id)
  AS churn_rate_180d
FROM active_prev LEFT JOIN active_curr USING (customer_id);
```

- **Gotcha:** Churn is **purchase-based**, not engagement-based. A customer who browses every day but doesn't buy is "churned" by this definition. If you need engagement-churn, define a separate metric and add it here.

---

## GMV — Gross Merchandise Value

- **Definition:** Sum of `order_subtotal` (pre-tax, pre-shipping merchandise total) across orders with `status = 'completed'`.
- **Source:** `analytics_prod.core.fct_orders`

```sql
SELECT
  order_date,
  SUM(order_subtotal) AS gmv
FROM analytics_prod.core.fct_orders
WHERE order_date BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
  AND status = 'completed'
GROUP BY order_date;
```

- **Exclusions:** taxes, shipping, gift card top-ups (not "merchandise").
- **Gotcha:** GMV ≠ revenue. Revenue must reconcile with finance and includes refund subtraction. Use **net revenue** below for finance-aligned numbers.

## Net revenue

- **Definition:** GMV − refunds − returns − chargebacks, attributed to the **original order date** (not the refund date).
- Finance-aligned. Use this for any external-facing dollar number.

```sql
SELECT
  o.order_date,
  SUM(o.order_subtotal)
    - COALESCE(SUM(r.refund_amount), 0) AS net_revenue
FROM analytics_prod.core.fct_orders o
LEFT JOIN analytics_prod.core.fct_refunds r
  ON o.order_id = r.order_id
WHERE o.order_date BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
  AND o.status IN ('completed','refunded','partially_refunded')
GROUP BY o.order_date;
```

---

## AOV — Average Order Value

- **Definition:** `GMV / count(completed orders)` over the window.
- **Gotcha:** Customer-level AOV (`AVG(order_total) per customer`) is a different metric — don't confuse them.

```sql
SELECT
  SUM(order_subtotal) / COUNT(*) AS aov
FROM analytics_prod.core.fct_orders
WHERE order_date BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
  AND status = 'completed';
```

---

## Conversion rate (visitor → buyer, session-level)

- **Definition:** Share of sessions with a `purchase` event over sessions with at least one `page_view`.
- **Source:** `fct_page_events`

```sql
WITH sessions AS (
  SELECT
    session_id,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS converted
  FROM analytics_prod.core.fct_page_events
  WHERE event_date BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
    AND is_synthetic = FALSE
  GROUP BY session_id
)
SELECT AVG(converted) AS session_conversion_rate FROM sessions;
```

- **Gotcha:** Don't compute as `purchases / page_views` — that's events, not sessions, and it's wrong by a factor of "events per session."

---

## LTV — Lifetime Value (12-month)

- **Definition:** Sum of net revenue from a customer over the 12 months following their first completed order.
- **Cohort:** Customers whose `first_order_date` falls in the analysis window.

```sql
SELECT
  c.first_order_date,
  AVG(ltv.revenue) AS avg_ltv_12mo
FROM analytics_prod.core.dim_customers c
JOIN (
  SELECT
    o.customer_id,
    SUM(o.order_subtotal - COALESCE(r.refund_amount, 0)) AS revenue
  FROM analytics_prod.core.fct_orders o
  LEFT JOIN analytics_prod.core.fct_refunds r ON o.order_id = r.order_id
  JOIN analytics_prod.core.dim_customers c ON o.customer_id = c.customer_id AND c.is_current = TRUE
  WHERE o.order_date BETWEEN c.first_order_date AND DATE_ADD(c.first_order_date, 365)
    AND o.status IN ('completed','refunded','partially_refunded')
  GROUP BY o.customer_id
) ltv ON c.customer_id = ltv.customer_id
WHERE c.is_current = TRUE
  AND c.first_order_date BETWEEN DATE '2024-01-01' AND DATE '2024-01-31'
GROUP BY c.first_order_date;
```

- **Gotcha:** Truncated at 12 months — customers in the cohort who haven't reached month 12 yet must be excluded or the average is biased downward.

---

## Cart abandonment rate

- **Definition:** Sessions with `add_to_cart` but no `purchase` in the same session, over sessions with `add_to_cart`.

```sql
WITH carts AS (
  SELECT
    session_id,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS purchased
  FROM analytics_prod.core.fct_page_events
  WHERE event_date BETWEEN DATE '2025-01-01' AND DATE '2025-01-31'
    AND event_name IN ('add_to_cart', 'purchase')
    AND is_synthetic = FALSE
  GROUP BY session_id
  HAVING MAX(CASE WHEN event_name = 'add_to_cart' THEN 1 ELSE 0 END) = 1
)
SELECT 1 - AVG(purchased) AS cart_abandonment_rate FROM carts;
```

- **Gotcha:** This is **session-level**. Cross-session conversion (added to cart Monday, purchased Wednesday) needs a different definition — request it explicitly.

---

## Adding a new metric

When the user asks for a metric not in this file:

1. **Don't compute it yet.** Confirm the definition first.
2. Propose: numerator, denominator, source table, exclusions, gotchas.
3. After agreement, add it here using the template above.
4. Run Prompt 4 in `README.md` ("Extract a metric definition") if multiple definitions exist in the codebase.
