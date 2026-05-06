---
name: analysis-planner
description: Turn a fuzzy stakeholder question into a structured plan — clarifying questions, hypotheses, metrics, data sources, method, and deliverable shape — before any query runs.
when_to_invoke:
  - "Help me plan an analysis"
  - "Stakeholder asked for X, what should I do?"
  - "How should I approach this question?"
  - The user pastes a fuzzy ask in 1–2 sentences with unclear scope.
---

# Analysis Planner

The "ask clarifying questions before proceeding" pattern, codified. Most failed analyses fail at planning, not execution. This skill produces a one-page plan the user can sanity-check **before** the agent runs a single query.

## Inputs

- The stakeholder's ask, verbatim. The fuzzier the better — that's the point.
- Optional: any constraints (deadline, audience, expected deliverable form).

## Outputs

A short plan with five sections:

1. **Clarifying questions** — explicit, numbered, answerable.
2. **Hypotheses** — what you'd expect, what would surprise you.
3. **Metrics & tables** — exact metrics from `docs/metric-definitions.md` and tables from `docs/schema/`.
4. **Method** — the rough analytic approach (cohort, funnel, root-cause, A/B, model).
5. **Deliverable shape** — a markdown summary, Slack post, dashboard, slides, etc. Estimated effort.

## Procedure

1. **Read** `docs/business-info.md`, `docs/metric-definitions.md`, and `docs/dashboards.md`. Check whether an existing dashboard already answers the question — if yes, point to it before planning anything else.
2. **List clarifying questions.** Default checklist: time range, audience, surfaces (web/app), regions, exclusions (employees, test accounts, internal traffic, refunded orders), and the deliverable's audience (exec vs PM vs Slack channel). Do not proceed without answers — flag them and stop.
3. **Form hypotheses.** State the team's prior expectation in one sentence. State what would surprise you. If you have no prior, say so.
4. **Pick metrics + tables.** Reference exact metric names from `docs/metric-definitions.md` and exact table names from `docs/schema/`. If a metric isn't documented, flag it and propose a definition (do not invent and run).
5. **Pick the method.** One of: descriptive (counts/trends), cohort, funnel, root-cause, experiment analysis, predictive. Justify in one sentence.
6. **Sketch the deliverable.** Output shape (table / chart / paragraph), audience, length budget, where it'll live. Estimate effort in hours, half-days, or days.
7. **Stop.** Hand the plan to the user and wait for sign-off before any execution.

## Big-data / safety guards

- **Do not run any query during planning.** Planning is read-only.
- **If the deliverable is exec-facing**, flag the reconciliation requirement: numbers must match `docs/metric-definitions.md` and (for revenue) the finance-aligned `net revenue` definition.
- **If the analysis window overlaps a calendar event** in `docs/business-info.md` (BFCM, holiday peak, etc.), call it out in the plan — it changes the interpretation.

## Worked example

> User: "Why did our 7-day retention drop last week?"

The plan:

1. **Clarifying questions:**
   - Which surfaces? (default: all)
   - Which segments? (region, channel, lifecycle stage — or all?)
   - "Last week" = which exact 7-day cohort? Day-0 dates of `2025-01-06` through `2025-01-12` matched against `D7` of `2025-01-13` through `2025-01-19`?
   - Compare against what baseline? Prior week? Trailing 4-week average?
   - Audience for the deliverable? (Slack post for PM vs full deck)

2. **Hypotheses:**
   - **Prior:** D7 retention is stable week-over-week (±0.5 pts). A real drop > 1 pt is unusual.
   - **Suspects (in priority order):** mix shift in acquisition channel (paid social influx skews lower-quality cohorts); a release on `2025-01-08` broke push notifications on Android (check release notes); BFCM cohort tail effect (deal-driven first-time buyers retain worse).

3. **Metrics & tables:**
   - **Metric:** `7-day retention (D7)` — definition in `docs/metric-definitions.md`.
   - **Tables:** `analytics_prod.core.fct_page_events` (qualifying events), `analytics_prod.core.dim_customers` (channel, segments).

4. **Method:** Root-cause investigation. Run the metric for the affected window, decompose by `surface`, `region`, `first_touch_channel`, `lifecycle_stage`. Check for change-points around 2025-01-08. Use `root-cause-investigator` skill.

5. **Deliverable:** Slack thread for PM → 4 chart png + 6-bullet summary. ~3 hours.

## What to add to docs/ if missing

- If a clarifying question keeps coming up across analyses → add the standing assumption to `docs/business-info.md` ("default exclusions for company-wide metrics").
- If a metric isn't documented → propose adding to `docs/metric-definitions.md` (use Prompt 4 in the README).
