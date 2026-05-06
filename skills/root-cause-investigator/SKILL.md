---
name: root-cause-investigator
description: When a metric moves unexpectedly, decompose it — by dimension, cohort, change-point — to localize the cause before proposing a hypothesis.
when_to_invoke:
  - "Metric X dropped — why?"
  - "What caused the spike in Y?"
  - "Why is conversion down?"
  - "Investigate the recent change in <metric>"
---

# Root Cause Investigator

When a metric moves, **decompose before you hypothesize.** The goal is to localize the move to a slice of the data — surface, region, channel, segment, time-of-day, cohort — so the hypothesis space is small enough to test.

## Inputs

- **Metric** that moved (must be defined in `docs/metric-definitions.md`).
- **The move**: window, magnitude, expected baseline.
- Optional: known recent changes (releases, campaigns, infra).

## Outputs

- A **decomposition** showing where the move concentrates.
- A **change-point** estimate (when did the metric break?).
- A **ranked list of hypotheses** with the evidence for/against each.
- Suggested next step (deeper dive on the leading hypothesis, or hand to the owning team).

## Procedure

1. **Confirm the move is real.** Is the trailing window noisy enough that this magnitude could be normal? Compare to trailing 4-week stddev. If within 1σ of the trailing mean, **stop** — it's noise. Tell the user.
2. **Localize in time.** Plot the metric daily for the trailing 30 days. Identify the change-point — is it a step (sudden), a slope (gradual), or a single outlier? Different shapes → different hypotheses.
3. **Decompose by dimension.** Run the same metric broken out by, in this order:
   - **Surface** (web / ios / android)
   - **Region**
   - **Acquisition channel** (`first_touch_channel`)
   - **Lifecycle stage**
   - **New vs repeat buyers** (`is_first_order`)
   For each: does the move concentrate in a slice (>2× the overall move) or is it broad-based?
4. **Decompose by cohort.** Is the move driven by a recent cohort behaving worse, or by older cohorts shifting? Cohort analysis localizes "new acquisition got worse" vs "existing customers stopped engaging."
5. **Check release / campaign / infra calendar:**
   - Releases on or near the change-point (`git log` on relevant code, deployment logs).
   - Marketing campaigns that started/ended (check `stg_marketing__campaigns` in `docs/external-tables/`).
   - Pipeline incidents (was upstream data wrong? — check `docs/pipelines.md` for outages).
6. **Check the data itself** before blaming the product. Run `data-quality-checker` against the underlying tables. **Half of "metric moved" investigations resolve as data issues, not real moves.**
7. **Rank hypotheses** by evidence weight. State each as: "<hypothesis> — supported by <evidence>; refuted by <evidence>; would be confirmed by <test>."
8. **Stop before drilling on the leading hypothesis** unless the user asks. The decomposition itself is usually the deliverable.

## Big-data / safety guards

- Apply partition filters on every decomposition query — no global table scans.
- Apply your metric's standard exclusions (synthetic events, test accounts, employees, cancelled orders) per `docs/metric-definitions.md`. Don't assume column names — look them up.
- For new-buyer decompositions, filter to first-order customers using whatever flag your team tracks (check `docs/schema/` for your customers/orders tables).

## Worked example

> User: "Why did D7 retention drop from ~28% to ~25% last week?"

1. **Confirm real.** Trailing 4-week mean 28.1%, stddev 0.6 pts. Drop to 25% is ~5σ — real.
2. **Time shape.** Daily D7 by D0-date for trailing 30 days: stable at ~28% until D0=2025-01-08, then steps to ~25% and stays. Step change, not gradual. Change-point ≈ 2025-01-08.
3. **Decompose:**
   - **Surface:** web 28→27 (-1 pt); ios 29→24 (-5 pts); android 27→22 (-5 pts). **Move concentrates in app surfaces.**
   - **Region:** uniform across regions; rules out regional outage.
   - **Channel:** mostly uniform; paid_social slightly worse.
   - **Lifecycle:** repeat-buyer retention unchanged; first-time-buyer retention dropped 7 pts. **Move concentrates in first-time buyers.**
4. **Cohort.** Cohorts active on 2025-01-08+ are worse; older cohorts unchanged. Confirms the issue is *new* engagement, not aging.
5. **Release calendar.** App release v4.21.0 deployed 2025-01-08 — release notes mention push-notification refactor.
6. **Data check.** `fct_page_events` row counts normal; no schema drift; `is_synthetic` not unusual. Data is fine.
7. **Hypotheses (ranked):**
   - **H1: App v4.21.0 broke push notifications for first-time buyers** — supported by surface concentration (apps), lifecycle concentration (FTB), timing (2025-01-08), release notes mention push refactor. Refuted: nothing yet. **Confirm by:** comparing push-send and push-open events between 2025-01-07 and 2025-01-08 in `fct_app_events`; pinging the app team.
   - **H2: BFCM-tail cohort effect** — supported by FTB concentration. Refuted by surface concentration (BFCM was platform-agnostic), and timing (BFCM cohorts would show drift earlier). Lower probability.
   - **H3: Data issue with the metric itself** — refuted by row-count checks and surface-specific concentration.
8. **Recommended next step:** confirm H1 with the app team. Don't ship more analysis until H1 is verified.

## What to add to docs/ if missing

- A new dimension that should be in the standard decomposition (e.g., "always break out by `signup_surface`" if it surfaces issues repeatedly) → add to this skill's procedure list.
- A confirmed root cause that's product-relevant → add to `docs/experiment-registry.md` (if it was triggered by a feature change) or `docs/pipelines.md` (if it was a data issue).
