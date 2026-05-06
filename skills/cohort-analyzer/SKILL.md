---
name: cohort-analyzer
description: Build a cohort retention or conversion analysis — define the cohort, the time windows, and produce the retention triangle plus chart-ready output.
when_to_invoke:
  - "Show me retention by cohort"
  - "What's the retention curve for new buyers?"
  - "How does cohort X compare to cohort Y?"
  - "Build a retention triangle"
---

# Cohort Analyzer

Cohort analyses are the right tool when you want to compare groups defined by **when they joined** (or "started doing X"), separating the maturation effect from a true behavioral change.

## Inputs

- **Cohort definition.** First action (e.g., `first_order_date`, `signup_ts::date`) and any cohort dimension (channel, region).
- **Behavior to track** (retention, repeat purchase, conversion to a downstream event).
- **Time windows.** Day, week (recommended for noisy daily series), or month.
- **Number of periods.** Default 12 weeks.

## Outputs

- A retention triangle table (cohort × period_offset → retained share).
- A chart-ready long-format DataFrame.
- A summary sentence per cohort: "Cohort 2024-12-01 has 68% W1 retention, 42% W4."

## Procedure

1. **Confirm the cohort definition** with the user. The single most common mistake is conflating "cohort by signup date" with "cohort by first-order date" — these produce different numbers. State which one you're using.
2. **Cap the analysis window.** A cohort that started yesterday cannot have W4 retention yet. Exclude cohorts that haven't reached the largest period offset, or mark them as "incomplete" in the output.
3. **Build the cohort table:** for each customer, their cohort label + their first action date.
4. **Build the activity table:** for each customer, the dates of their tracked activity.
5. **Join + bucket** activity to period offsets (`period_offset = floor((activity_date - cohort_date) / period_length)`).
6. **Aggregate:** distinct customers per (cohort, period_offset) over distinct customers per cohort = retention rate.
7. **Sanity-check:**
   - Period 0 retention should be 100% (everyone in the cohort is active by definition on day 0). If not, your cohort or activity definition is wrong.
   - Retention should be monotonically non-increasing over period offsets. If it isn't, you have a bug — check your events table for known duplicate gotchas (see `docs/schema/` for the table you're using).

## Big-data / safety guards

- For weekly cohorts with > 6 months of history, **always partition-filter** your events table to the analysis window, not the full table.
- Cohorts smaller than **N=50** are noisy and should not be charted alone. Either combine with adjacent cohorts or annotate the chart.
- Use deterministic seeds when sampling cohorts for visualization.

## Worked example

> User: "Build a 12-week retention triangle for new buyers in November 2024, broken out by acquisition channel."

1. **Cohort definition:** customers whose `first_order_date` is in November 2024. Cohort label = `date_trunc('week', first_order_date)`.
2. **Behavior:** any qualifying event in `fct_page_events`.
3. **Window:** 12 weeks. November 2024 cohorts have ≥ 8 weeks of history as of late January 2025 — flag W9–W12 as "incomplete" for the latest cohorts.

```sql
WITH cohort AS (
  SELECT
      c.customer_id
    , date_trunc('week', c.first_order_date) AS cohort_week
    , c.first_touch_channel
  FROM analytics_prod.core.dim_customers c
  WHERE c.is_current = TRUE
    AND c.first_order_date BETWEEN DATE '2024-11-01' AND DATE '2024-11-30'
    AND c.is_test_account = FALSE
    AND c.is_employee = FALSE
),
activity AS (
  SELECT DISTINCT
      pe.customer_id
    , date_trunc('week', pe.event_date) AS activity_week
  FROM analytics_prod.core.fct_page_events pe
  WHERE pe.event_date BETWEEN DATE '2024-11-01' AND date_add(DATE '2024-11-30', 12*7)
    AND pe.event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
    AND pe.is_synthetic = FALSE
    AND pe.customer_id IS NOT NULL
)
SELECT
    c.cohort_week
  , c.first_touch_channel
  , datediff(a.activity_week, c.cohort_week) / 7 AS week_offset
  , COUNT(DISTINCT a.customer_id) * 1.0 / COUNT(DISTINCT c.customer_id) OVER (
      PARTITION BY c.cohort_week, c.first_touch_channel
    ) AS retained_share
FROM cohort c
LEFT JOIN activity a ON a.customer_id = c.customer_id AND a.activity_week >= c.cohort_week
GROUP BY c.cohort_week, c.first_touch_channel, week_offset, c.customer_id
;  -- This is illustrative; in practice compute denominator separately and join (cleaner pattern).
```

> Note: the query above is illustrative — for production cohort tables, compute denominators per cohort in a separate CTE and join, rather than mixing aggregations and window functions.

4. Output (truncated):

```
cohort_week  channel        W0    W1    W2    W3    W4
2024-11-04   organic       100%  62%   48%   41%   37%
2024-11-04   paid_search   100%  58%   43%   36%   32%
2024-11-04   paid_social   100%  51%   34%   26%   21%
...
```

5. Hand to `plotting-graphs` for the retention heatmap. Hand to `analysis-summarizer` for the writeup.

## What to add to docs/ if missing

- A canonical cohort definition the team uses (e.g., "default cohort = first_order_date") → add to `docs/metric-definitions.md` under a new "Cohort definitions" section.
- An issue with the cohort table itself (e.g., `first_order_date` lag) → already noted in `docs/schema/dim_customers.md`; reinforce if it bit you.
