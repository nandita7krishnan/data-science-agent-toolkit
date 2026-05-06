# End-to-end walkthrough — "Why did D7 retention drop last week?"

A worked example showing how `docs/`, `skills/`, and the workflow fit together. Read this top-to-bottom to see the full loop in one place.

The fuzzy ask: a PM pings the DS channel.

> "Hey, our D7 retention dropped last week — like 28% to 25%. Can you take a look?"

What follows is what the agent (with this repo loaded) does, step-by-step. The procedure traces through specific docs, specific skills, and the same patterns described in the README workflow.

---

## 0. Read the closest `AGENTS.md`

The agent is being asked an analysis question — it'll be working in `analyses/`. So it reads:

1. `AGENTS.md` (root) — tech stack, conventions, guardrails.
2. `analyses/AGENTS.md` — notebook conventions, reproducibility checklist.

The root file says: "Ask clarifying questions before proceeding when the request is ambiguous." That's exactly what's about to happen.

## 1. Plan before doing — invoke `analysis-planner`

The agent reads `skills/analysis-planner/SKILL.md` and runs through its procedure:

1. **Read** `docs/business-info.md`, `docs/metric-definitions.md`, `docs/dashboards.md`. Discovery: the "Funnel Health" and "Cohort Retention" dashboards already track retention but at a different grain (cohort × channel, not week-over-week). They don't answer this exact question. Proceed.
2. **Clarifying questions** for the PM:
   - Which surfaces? Default: all (`web`, `ios`, `android`).
   - Which segments? Break out by all of `surface`, `region`, `first_touch_channel`, `lifecycle_stage`?
   - "Last week" — exact dates? Comparing to what baseline (prior week, trailing 4-week mean)?
   - Audience for the deliverable — Slack thread for you, or a writeup for the broader team?
3. **Hypotheses (going in):**
   - **Prior:** D7 retention is stable week-over-week (±0.5 pts). A 3-pt drop is unusual.
   - **Suspects in priority order:** acquisition mix shift, app release on `2025-01-08`, BFCM-cohort tail effect, data issue.
4. **Metrics & tables:**
   - Metric: `7-day retention (D7)` per `docs/metric-definitions.md`. Exact formula will be reused.
   - Tables: `analytics_prod.core.fct_page_events` (qualifying events), `analytics_prod.core.dim_customers` (segments).
5. **Method:** Root-cause investigation — invoke `root-cause-investigator` skill after the plan is approved.
6. **Deliverable:** Slack thread to PM with 4 chart attachments + a 6-bullet summary. ~3 hours.

The agent posts this plan to the PM and **stops.** No queries yet.

> *PM responds: "All surfaces. Break out by surface, region, channel, lifecycle. 'Last week' = D0 dates 2025-01-06 to 2025-01-12 vs trailing 4-week. Slack thread is fine."*

---

## 2. Validate the source data — invoke `data-quality-checker`

Before computing the metric, sanity-check the source. The agent reads `skills/data-quality-checker/SKILL.md` and runs the checklist against `fct_page_events`:

- **Freshness:** `MAX(event_date) = current_date()`, latest hour partition loaded. Within SLA. ✓
- **Row count trend:** last 14 days, all within ±2σ of trailing-7-day mean. ✓
- **Duplicates:** spot-check on `event_id` for `event_date = 2025-01-08` (the suspected change-point) — clean. ✓
- **Synthetic / bot rate:** stable. ✓
- **Schema:** matches `docs/schema/fct_page_events.md`. ✓

The data is clean. Move on.

---

## 3. Compute the metric — invoke `metric-calculator`

The agent reads `skills/metric-calculator/SKILL.md` and the D7 retention entry in `docs/metric-definitions.md`. It copies the canonical formula and substitutes the date variables, adding a join to `dim_customers` for `is_test_account = FALSE AND is_employee = FALSE`.

It then runs `query-reviewer` on the SQL (per the procedure):

- ✓ partition filters on `event_date`
- ✓ `is_synthetic = FALSE` filter
- ✓ `is_current = TRUE` on `dim_customers`
- ✓ no `SELECT *`

Query passes. Execute. Result for `D0 = 2025-01-08`:

- Cohort size: 1,127,402 customers
- D7 retained: 285,341
- D7 retention: **25.3%**
- Trailing 4-week mean: 28.1% (stddev 0.6 pts)
- 4.7σ below trailing mean. **Real drop, not noise.**

---

## 4. Decompose — invoke `root-cause-investigator`

The agent reads `skills/root-cause-investigator/SKILL.md` and runs the procedure:

### Time shape

Daily D7 retention by D0-date, trailing 30 days. The agent runs `metric-calculator` per day then plots via `plotting-graphs` skill (palette `Lavender Earth`, focus `#331E38` Midnight Violet).

```
Result: stable at ~28% from 2024-12-15 through 2025-01-07,
        steps to ~25% on D0 = 2025-01-08, flat at ~25% after.
        Step change. Change-point: 2025-01-08.
```

### Decompose by surface

Same metric, broken out by `surface`:

| Surface | Pre (D0=2025-01-01) | Post (D0=2025-01-08) | Δ |
|---|---|---|---|
| web | 28.4% | 27.6% | −0.8 |
| ios | 29.1% | 24.0% | **−5.1** |
| android | 27.0% | 22.1% | **−4.9** |

The drop concentrates in app surfaces.

### Decompose by region

Roughly uniform. Rules out a regional outage / localized event.

### Decompose by channel

Slightly worse on `paid_social`, but trend is broad-based. Doesn't localize.

### Decompose by lifecycle stage

| Lifecycle | Pre | Post | Δ |
|---|---|---|---|
| first_time_buyer | 22.4% | 15.3% | **−7.1** |
| repeat_buyer | 41.2% | 40.9% | −0.3 |
| vip | 58.0% | 57.5% | −0.5 |

The drop **concentrates** in first-time buyers.

### Cohort lens

Cohorts active on 2025-01-08 onward show the worse retention. Older cohorts unchanged. Confirms the issue is **new engagement dropping**, not aging.

### Calendar / release / data check

- **Releases:** the agent searches release notes / git logs for app commits near 2025-01-08. Finds: app v4.21.0 deployed 2025-01-08, release notes mention "push notification refactor."
- **Marketing:** no campaign starts/ends on 2025-01-08 per `stg_marketing__campaigns`.
- **Data quality:** already checked. Clean.

### Hypothesis ranking

| Rank | Hypothesis | Supported by | Refuted by | Confirm with |
|---|---|---|---|---|
| 1 | App v4.21.0 broke push notifications for first-time buyers | surface concentration (apps), lifecycle concentration (FTB), timing (2025-01-08), release notes mention push | nothing yet | App team — pull push-send and push-open events from `fct_app_events` for 2025-01-07 vs 2025-01-08+ |
| 2 | BFCM-cohort tail effect | FTB concentration | surface concentration is platform-agnostic for BFCM; timing is wrong | comparing FTB cohorts pre- and post-BFCM (would have shown earlier) |
| 3 | Data issue with the metric | (nothing) | row counts clean, schema fine, synthetic stable | already refuted in step 2 |

H1 has overwhelming evidence. **Recommend confirming H1 with the app team before more analysis.**

---

## 5. Visualize — invoke `plotting-graphs`

The agent generates the charts using `skills/plotting-graphs/SKILL.md` rules: `Lavender Earth` palette, focus on iOS/Android, muted on web, change-point line at 2025-01-08, direct labels, sentence-case title.

Saved to `reporting/figures/d7-retention/retention-drop-jan-2025-v1.{png,svg}`.

Alt-text:
> "D7 retention dropped from ~28% to ~25% on 2025-01-08, concentrated in iOS (−5 pts) and Android (−5 pts) and in first-time buyers (−7 pts); coincides with the app v4.21.0 release."

---

## 6. PII check — invoke `pii-scrubber`

The output is **aggregate** (rates, segment counts ≥ 25). No customer-level data, no `customer_id` lists, no Confidential numbers. ✓

---

## 7. Summarize — invoke `analysis-summarizer`

Audience = PM Slack channel. Mrkdwn formatting, headline first.

```
*D7 retention dropped from 28% → 25% last week — concentrated in iOS/Android first-time buyers, change-point on 2025-01-08.*

• *Where:* iOS −5 pts, Android −5 pts, Web flat. First-time buyers −7 pts; repeat buyers and VIPs unchanged.
• *When:* clean step on 2025-01-08; flat before, flat after.
• *Why (leading hypothesis):* App v4.21.0 deployed 2025-01-08 refactored push notifications. Hypothesis to verify with app team.
• *Why not data:* row counts and synthetic-event filters look clean.

*Method:* trailing 30 days of `fct_page_events` D7 retention, decomposed by surface / region / channel / lifecycle / cohort. Standard exclusions (synthetic, internal traffic, anonymous, test accounts).

*Caveats:* hypothesis not yet confirmed against app push delivery data. Effect could be larger if push-send events are also being dropped.

*Next step:* @app-team — can you confirm push delivery rates for first-time buyers on iOS/Android since 2025-01-08?
```

Charts attached.

---

## 8. Self-improvement protocol

Did anything in this loop reveal a gap in the docs?

- **The `dim_customers` "no `is_current` filter" gotcha** is already in the schema doc — no edit needed.
- **The cohort-tail effect of BFCM** isn't mentioned anywhere as a candidate hypothesis. Propose adding to `docs/business-info.md` under "Calendar — periods that distort the data": "BFCM-cohort tail: deal-driven first-time buyers retain ~5 pts worse than baseline through their first ~6 weeks; affects retention numbers through mid-January."
- **No new skill needed** — the existing skills covered the workflow cleanly.

The agent posts the proposed `docs/business-info.md` edit as a final note: "Want me to commit this addition?"

---

## What this walkthrough demonstrates

- **Hierarchical AGENTS.md** — the agent read root + `analyses/` (different scope, different rules).
- **Progressive disclosure** — `docs/` files were read on-demand, not pre-loaded. The agent only opened the schema doc for `fct_page_events` after deciding to query it.
- **Skills as procedures** — eight skills invoked in order (`analysis-planner`, `data-quality-checker`, `metric-calculator`, `query-reviewer`, `root-cause-investigator`, `plotting-graphs`, `pii-scrubber`, `analysis-summarizer`). No improvising the procedure.
- **Canonical definitions** — D7 retention copied verbatim from `docs/metric-definitions.md`. No improvised formulas.
- **Self-improvement** — at the end, the agent surfaces a doc gap (BFCM-cohort tail effect) for the user to merge.

That's the whole loop. Real questions, on real ABC Business tables, using real skills, ending with a Slack post and a doc edit. **No magic.** Just docs that capture what the team knows, plus skills that capture how the team works.
