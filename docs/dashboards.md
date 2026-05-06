# Dashboards

Catalog of existing dashboards. **Check here before building a new analysis** — odds are, the answer already exists.

> Forking? Replace with your real dashboards. Keep the columns (Owner, Audience, Question Answered, Refresh, URL). Empty rows are fine; the discipline is "if you build a dashboard, log it here."

## Quick lookup

| Dashboard | Owner | Audience | Question it answers | Refresh | Link |
|---|---|---|---|---|---|
| **Daily KPIs** | DS | Exec, all-hands | "What did the business do yesterday?" — DAU, GMV, AOV, CR | 06:00 UTC daily | _< replace with your BI URL >_ |
| **Weekly Business Review (WBR)** | DS + Finance | Exec | The full WBR pack — revenue, margin, growth, cohorts | Manual, Mon AM | _< url >_ |
| **Marketing Performance** | Marketing Analytics | Marketing, DS | Channel ROI, attribution, CAC trends | Daily 07:00 UTC | _< url >_ |
| **Funnel Health** | DS | Product, PMs | Per-step conversion in the purchase funnel; alert on drops | Hourly | _< url >_ |
| **Cohort Retention** | DS | Product | New-buyer retention by cohort × channel | Weekly Mon | _< url >_ |
| **Pipeline Health** | Data Eng | Data team | Job runs, SLA misses, freshness | Real-time | _< url >_ |
| **Experiment Watcher** | Experimentation Platform | DS, PM | Live A/B tests — assignment counts, primary metric, sig | Hourly | _< url >_ |
| **Inventory & Sell-through** | Catalog Analytics | Merchandising, Ops | Sell-through rate by SKU; out-of-stock impact on GMV | Daily 08:00 UTC | _< url >_ |

## How to use this list

1. **Before scoping a new analysis**, scan the "Question it answers" column. If a dashboard already covers it, *use the dashboard*. Send the link, don't reproduce the work.
2. If the dashboard answers ~80% of the question and you need a one-off cut, *filter the dashboard*. Most BI tools let you add a temporary filter without forking.
3. If the question is structurally new (different grain, time range, dimensions), build the analysis. Then **add the new dashboard here** if it's going to be reused.
4. If a dashboard is **stale or wrong**, ping the owner. Don't quietly build a parallel one — you're creating a second source of truth.

## Adding a dashboard to this list

When you build a dashboard that's going to recur (anything you'd link to twice), add a row above with:

- **Dashboard name** — what it's called in the BI tool
- **Owner** — person or team that maintains it
- **Audience** — who it's built for
- **Question it answers** — one sentence, the actual question
- **Refresh** — schedule
- **Link** — direct URL

## Deprecation

When a dashboard is replaced or no longer maintained, mark it with `**[DEPRECATED YYYY-MM-DD → see <new dashboard>]**` rather than deleting the row. Stakeholders who bookmarked the old URL need to find the redirect.

## Common dashboards we *don't* have (yet)

Tracking gaps so we don't keep getting asked the same question without building the answer:

- Channel-level cart abandonment by surface (web vs app)
- LTV by acquisition cohort × first-touch channel
- Refund / chargeback rate by SKU (currently in finance ledger only)

> If you build any of these, move it up into the catalog.
