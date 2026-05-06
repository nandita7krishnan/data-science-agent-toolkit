---
name: metric-calculator
description: Calculate a metric using the team's exact canonical definition from docs/metric-definitions.md — never improvise.
when_to_invoke:
  - "What's our DAU / WAU / MAU / retention / GMV / AOV / churn / LTV / conversion rate?"
  - "Calculate <metric> for <window>"
  - The user asks for any metric defined in docs/metric-definitions.md.
---

# Metric Calculator

Calculate a metric the team has already defined. The whole job is **don't invent.** Read the canonical definition, use the canonical formula, validate against the canonical gotchas.

## Inputs

- Metric name (must match an entry in `docs/metric-definitions.md`).
- Time window (start date, end date).
- Optional: dimensions to break out by (e.g., region, surface, channel).
- Optional: additional filters (e.g., specific customer segment).

## Outputs

- The number(s).
- The exact query that produced them (for reproducibility).
- Any caveats from the metric's "Gotchas" section that apply.

## Procedure

1. **Read `docs/metric-definitions.md` and find the metric.** If it isn't there, **stop** — ask the user for the definition or run Prompt 4 ("Extract a metric definition") in `README.md`.
2. **Copy the formula verbatim.** Do not edit the SQL except to substitute time-window literals and add the requested breakouts.
3. **Apply the metric's exclusions.** They're in the metric definition — don't drop any.
4. **Apply user-level exclusions** unless told otherwise. Common patterns: test accounts, internal employees, bots. Check `docs/data-classification.md` or `docs/metric-definitions.md` for your team's exclusion columns — don't assume column names.
5. **Run `query-reviewer`** on the resulting SQL.
6. **Execute** (assuming the user has approved the query — don't auto-run).
7. **Sanity-check the output** against `docs/business-info.md`:
   - Is the window in a calendar period (BFCM, holiday) that distorts the number?
   - Is the level of magnitude plausible (within 2× of trailing values for stable metrics)?
8. **Report** with: the number, the query, the caveats, the comparison context (e.g., "vs prior week, vs trailing 4-week average").

## Big-data / safety guards

- **Don't compute a metric that isn't in `docs/metric-definitions.md`.** If asked, propose a definition and stop.
- **Don't substitute "close-enough" tables.** If `metric-definitions.md` says use `fct_page_events`, don't use `fct_orders` because it's smaller.
- **Don't aggregate across surfaces / regions** unless the metric definition says to. Default to the documented breakout, ask before flattening.
- **Reconcile with `net revenue` for any external-facing dollar number.** GMV ≠ revenue.

## Worked example

> User: "What was our 7-day retention for the cohort active on 2025-01-08?"

1. Read `docs/metric-definitions.md` → "7-day retention (D7)". Definition: distinct customers active on D0 *and* on D0+7. Exclusions: bots, internal IPs, synthetic events, anonymous (`customer_id IS NULL`).
2. Copy the formula. Substitute `D0 = 2025-01-08`, `D7 = 2025-01-15`.
3. Apply user-level exclusions per your team's convention (check `docs/metric-definitions.md` for the D7 exclusion list).
4. Pass `query-reviewer`. Execute.

```sql
WITH d0 AS (
  SELECT DISTINCT pe.customer_id
  FROM analytics_prod.core.fct_page_events pe
  JOIN analytics_prod.core.dim_customers c
    ON c.customer_id = pe.customer_id AND c.is_current = TRUE
  WHERE pe.event_date = DATE '2025-01-08'
    AND pe.event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
    AND pe.is_synthetic = FALSE
    AND pe.customer_id IS NOT NULL
    AND c.is_test_account = FALSE
    AND c.is_employee = FALSE
),
d7 AS (
  SELECT DISTINCT pe.customer_id
  FROM analytics_prod.core.fct_page_events pe
  JOIN analytics_prod.core.dim_customers c
    ON c.customer_id = pe.customer_id AND c.is_current = TRUE
  WHERE pe.event_date = DATE '2025-01-15'
    AND pe.event_name IN ('page_view','product_view','add_to_cart','begin_checkout','purchase')
    AND pe.is_synthetic = FALSE
    AND pe.customer_id IS NOT NULL
    AND c.is_test_account = FALSE
    AND c.is_employee = FALSE
)
SELECT
    COUNT(DISTINCT d7.customer_id) * 1.0 / COUNT(DISTINCT d0.customer_id) AS d7_retention
FROM d0 LEFT JOIN d7 USING (customer_id);
```

5. Result: `27.4%`.
6. Sanity check: trailing 4-week D7 average is 28.1%. Drop of 0.7 pts; within normal week-over-week noise but at the edge — flag for the user. The cohort window does **not** overlap a calendar event in `docs/business-info.md`.

Final response:

> **D7 retention for the 2025-01-08 cohort: 27.4%.**
> Trailing 4-week average: 28.1%. Within typical noise but at the lower edge — confirm with the user if a follow-up is wanted (consider running `root-cause-investigator`).
> Caveat from `metric-definitions.md`: D7 = exactly 7 days later, not "anytime within 7 days." For "anytime within 7 days," use rolling D7.

## What to add to docs/ if missing

- A new metric requested → propose adding to `docs/metric-definitions.md`.
- A new exclusion the team agrees on → add to the metric's "Exclusions" section.
- A repeated sanity-check that catches issues → add as a "Validation" subsection on the metric.
