---
name: analysis-summarizer
description: Turn notebook output into a structured summary tailored to the audience — exec, Slack, email, or slide bullets — with the headline first.
when_to_invoke:
  - "Summarize this analysis"
  - "Write the writeup for this notebook"
  - "Turn this into a Slack post / email / exec summary"
  - "Format these results for <audience>"
---

# Analysis Summarizer

A good summary leads with **the answer**, not the journey. This skill produces a structured writeup matched to the audience: exec one-liner, Slack thread, email, or deck bullets.

## Inputs

- The analysis output (notebook, table of numbers, key findings).
- **Audience**: exec | PM | DS team | Slack channel | email distribution | slide deck.
- Optional: length budget (default: as terse as the audience tolerates).

## Outputs

- A summary in the right format with the right tone.
- Always includes: headline, key numbers, methodology sentence, caveats, recommended next step.

## Procedure

1. **Lead with the headline.** Single sentence. The number, the change, the implication. **Never** open with "I ran an analysis on..." — that's the journey, not the answer.
2. **Then 2-4 supporting bullets.** Each is a fact + interpretation, not a fact alone.
3. **Methodology sentence.** One line: data source(s), time window, exclusions. Enough for a peer to reproduce; not so much that exec readers stall.
4. **Caveats — explicit, not buried.** What could be wrong, what's not in scope, what would change the conclusion.
5. **Recommended next step.** What should the audience do? Decide, ask a follow-up question, run another analysis, ship the change?
6. **Match the format:**
   - **Exec / weekly review:** 3 bullets max + the chart. Numbers in plain English. No Spark SQL.
   - **PM:** ~6-10 bullets, includes one chart, methodology sentence, leading hypothesis.
   - **DS team:** full writeup with code/query references, segment breakouts, bootstrap CIs.
   - **Slack:** plain text or mrkdwn (no triple-backticks-in-codeblock issues), thread-friendly. Headline in bold. Chart as image attachment.
   - **Email:** subject line is the headline. Body = bullets + chart inline.
   - **Slide:** one number per slide, in 60+ pt type. Bullets are speaker notes, not on-slide text.
7. **Run `pii-scrubber`** if the output contains any customer-level data (per `docs/data-classification.md`).

## Big-data / safety guards

- **No raw PII** in any summary destined outside the DS team.
- **No `cost_of_goods_sold`, `gross_margin`, or `lifetime_value`** in any summary destined for non-internal audiences (Confidential per `docs/data-classification.md`).
- **No GMV-as-revenue** in any exec-facing summary. Use `net revenue` (or call out GMV explicitly).
- **No same-day marketing numbers** without the "subject to retroactive correction" caveat, per `docs/external-tables/stg_marketing__campaigns.md`.

## Worked example

> User: "Summarize the D7-retention drop investigation for the PM Slack channel."

Output (Slack mrkdwn):

```
*D7 retention dropped from 28% → 25% last week — concentrated in iOS/Android first-time buyers, change-point on 2025-01-08.*

• *Where:* iOS −5 pts, Android −5 pts, Web flat. First-time buyers −7 pts; repeat buyers unchanged.
• *When:* clean step on 2025-01-08; flat before, flat after.
• *Why (leading hypothesis):* App v4.21.0 deployed 2025-01-08 refactored push notifications. Hypothesis to verify with app team.
• *Why not data:* row counts and synthetic-event filters look clean.

*Method:* trailing 30 days of `fct_page_events` D7 retention, decomposed by surface / region / channel / lifecycle / cohort. Standard exclusions (synthetic, internal traffic, anonymous, test accounts).

*Caveats:* hypothesis not yet confirmed against `fct_app_events` push events. Effect could be larger than measured if push-send events are also being dropped.

*Next step:* @app-team — can you confirm push delivery rates for FTBs on iOS/Android since 2025-01-08? Happy to pair on the data.
```

## What to add to docs/ if missing

- An audience that keeps recurring (e.g., "monthly board update") with a specific format → add to this skill's procedure under step 6.
- A formatting convention that comes up (e.g., "always include effect size with CI in any number") → add to `docs/style-guide.md` under a new "Deliverables" section.
