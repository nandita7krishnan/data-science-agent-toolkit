---
name: funnel-analyzer
description: Build a step-by-step conversion funnel — drop-off attribution, time-to-convert, segmented breakouts — from event data.
when_to_invoke:
  - "Build a funnel from X to Y"
  - "Where do users drop off?"
  - "What's the conversion rate from step A to step B?"
  - "How long do users take between A and B?"
---

# Funnel Analyzer

A funnel is the right tool when there's an **ordered sequence of events** and the team wants to know per-step conversion and drop-off. Don't use it for unordered behaviors (use cohort analysis instead).

## Inputs

- **Ordered list of steps** as event names, e.g., `['product_view', 'add_to_cart', 'begin_checkout', 'purchase']`.
- **Time window.** Default: last 7 days for funnel-health monitoring; custom for one-off questions.
- **Funnel boundary**: session-level (default), customer-level, or fixed-window (e.g., "within 24 hours of step 1").
- **Optional segments** to break out by (surface, channel, region).

## Outputs

- Per-step counts and conversion rates (step `i` → step `i+1`).
- Time between steps (median, p95).
- Segment-level drop-off comparison (where does mobile drop off worst?).

## Procedure

1. **Confirm the boundary.** "How is `add_to_cart → purchase` measured?" — same session, same customer same day, within 24 hours? This **massively** changes the number.
2. **Build a per-(boundary-unit) flag table:** for each session (or customer-window), did each step occur?
3. **Sequence-check** if order matters. For "must happen in order," use `MIN(event_ts)` per step and require `step1_ts < step2_ts`. For "any order, all present," skip the sequence check.
4. **Step counts:** sum the flags. Step `i` count = boundary-units where step `i` flag = 1 *and* prior steps' flags = 1 (if order required).
5. **Conversion rates:** step `i+1` count / step `i` count. **Don't** report step counts as a single % of step 1 unless asked — the per-step conversion is more diagnostic.
6. **Time-to-convert** between adjacent steps: `MIN` of step i+1 ts − `MIN` of step i ts. Report median + p95.
7. **Segment breakouts.** Same query, grouped by the segment dimension.
8. **Sanity-check:** funnel must be monotonically non-increasing. If step `i+1` count > step `i` count, you have a bug (likely sequence or boundary).
9. Hand to `plotting-graphs` for the funnel chart and `analysis-summarizer` for the writeup.

## Big-data / safety guards

- Always filter your events table on the partition column and your team's synthetic/bot exclusion (check `docs/schema/` for the right column name).
- For "customer-level" or "fixed-window" funnels, the join can explode rows — `EXPLAIN` first if the window is > 7 days.
- App-surface purchase events are often lossy in event streams — for "% who purchased" steps, prefer joining to your orders table for the final step rather than relying on a purchase event.

## Worked example

> User: "Last week's web funnel from product_view → add_to_cart → begin_checkout → purchase, by region."

1. Boundary: session-level. Order: required.
2. Build session-level step flags (excluding synthetic, web only):

```sql
WITH session_steps AS (
  SELECT
      pe.session_id
    , pe.region
    , MIN(CASE WHEN event_name = 'product_view'   THEN event_ts END) AS pdp_ts
    , MIN(CASE WHEN event_name = 'add_to_cart'    THEN event_ts END) AS atc_ts
    , MIN(CASE WHEN event_name = 'begin_checkout' THEN event_ts END) AS chk_ts
    , MIN(CASE WHEN event_name = 'purchase'       THEN event_ts END) AS buy_ts
  FROM analytics_prod.core.fct_page_events pe
  WHERE pe.event_date BETWEEN date_sub(current_date(), 7) AND date_sub(current_date(), 1)
    AND pe.surface = 'web'
    AND pe.is_synthetic = FALSE
    AND pe.event_name IN ('product_view','add_to_cart','begin_checkout','purchase')
  GROUP BY pe.session_id, pe.region
)
SELECT
    region
  , COUNT(*) FILTER (WHERE pdp_ts IS NOT NULL)                                  AS s1_pdp
  , COUNT(*) FILTER (WHERE pdp_ts IS NOT NULL AND atc_ts > pdp_ts)              AS s2_atc
  , COUNT(*) FILTER (WHERE pdp_ts IS NOT NULL AND atc_ts > pdp_ts
                     AND chk_ts > atc_ts)                                       AS s3_checkout
  , COUNT(*) FILTER (WHERE pdp_ts IS NOT NULL AND atc_ts > pdp_ts
                     AND chk_ts > atc_ts AND buy_ts > chk_ts)                   AS s4_purchase
FROM session_steps
GROUP BY region
ORDER BY region;
```

3. Compute step-to-step conversion downstream (in Python or SQL):
   - `cr_atc = s2_atc / s1_pdp`
   - `cr_checkout = s3_checkout / s2_atc`
   - `cr_purchase = s4_purchase / s3_checkout`

4. Sanity check: each step's count is ≤ the prior step's. Monotonic. Pass.

5. Hand off to `plotting-graphs` (funnel chart, by region) and `analysis-summarizer` (the "where do we lose people" narrative).

## What to add to docs/ if missing

- The team's canonical funnel boundary (session vs customer-day) → add to `docs/metric-definitions.md` under a new "Funnel conventions" section.
- A new event to include in the canonical funnel → add to `docs/business-info.md` (funnel definition) and `docs/schema/fct_page_events.md`.
