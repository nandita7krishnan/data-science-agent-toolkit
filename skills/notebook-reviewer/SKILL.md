---
name: notebook-reviewer
description: Fresh-eyes review of an analysis notebook — hidden assumptions, missing CIs, sample-size issues, p-hacking risk, missing seed, cell ordering, missing data lineage — outputs ready-to-paste review comments.
when_to_invoke:
  - "Review this notebook"
  - "Critique this analysis"
  - "Is this analysis sound?"
  - User asks for a second-opinion review on a deliverable.
---

# Notebook Reviewer

Apply the "fresh agent reviews the implementer's work" pattern to data analyses. The reviewer is the same model with **no prior context** — it sees the notebook for the first time, like a stranger, and catches what the author was too close to see.

## Inputs

- Path to the notebook (`.ipynb`) or a rendered view of it.
- Optional: the underlying question (helps reviewer judge whether the notebook actually answers it).

## Outputs

- A list of review comments, each tagged by severity:
  - **Block** — can't ship until fixed (wrong number, broken logic, regulatory issue).
  - **Major** — methodologically suspect, must be addressed.
  - **Minor** — would improve clarity / robustness.
  - **Nit** — style, formatting.
- A summary verdict: **Ship-ready / Needs revision / Re-do.**

## Procedure

1. **Read the notebook top to bottom**, no skimming. Treat every cell as suspect until verified.
2. **Cell-order check.** Are there hidden state issues? A cell that uses a variable defined later, or one that re-runs out-of-order, breaks reproducibility. If you see `Out[5]` after `Out[12]` in execution numbering, flag it.
3. **Restart-and-Run-All check.** If you can re-run, do. If you can't, look for the giveaways: undefined variables, stale results, hard-coded numbers that don't match recomputed numbers.
4. **Methodology checklist:**
   - **Is the question stated?** A notebook without a stated question is a script.
   - **Time window declared?** And is it sane (no overlap with calendar events from `docs/business-info.md` without acknowledgement)?
   - **Exclusions applied?** `is_synthetic = FALSE`, `is_test_account = FALSE AND is_employee = FALSE`, `is_current = TRUE` on SCD2 dims.
   - **Metric definitions match `docs/metric-definitions.md`?** Spot-check at least one.
   - **Sample sizes** stated and adequate? Anything < 50 per cell of a breakout shouldn't be charted on its own.
   - **Confidence intervals** present on any reported difference or rate? A point estimate without a CI is a vibe.
   - **Multiple comparisons:** if the notebook tests 10 segments, was the significance threshold corrected (Bonferroni, BH)? "I found a significant slice" after 10 tries is p-hacking.
   - **Confounders addressed?** If comparing two groups without random assignment, what could explain the difference besides the cause being claimed?
   - **Seed set** per `docs/environment.md`?
   - **Data lineage in the writeup?** What tables, what version (today's data ≠ yesterday's data)?
5. **Reproducibility:**
   - All file paths relative to repo root.
   - No `pd.read_csv("/Users/me/Desktop/...")` paths.
   - All env vars loaded from `.env` (per `docs/environment.md`).
   - All randomized steps seeded.
6. **Charts** (`plotting-graphs` rules):
   - Do they have titles, subtitles ("so what"), and axis labels?
   - Sorted bars, zero baseline?
   - Annotated with the one number to take away?
7. **Conclusions:**
   - Match the data shown? (Common failure: chart shows "Android worse" but the conclusion claims "iOS worse.")
   - Caveats present and honest?
   - Recommended next step actionable, or wishy-washy?
8. **Output review comments** as a numbered list. Each comment is **paste-ready** — short, specific, and actionable. Don't write an essay.

## Big-data / safety guards

- **Don't re-run** queries the notebook contains as part of review (cost). Just read the queries.
- If any cell appears to embed PII (raw `email`, `customer_id` lists), call **Block** and require `pii-scrubber` before any further sharing.

## Worked example

> Reviewer was given `analyses/2025-01/d7-retention-drop.ipynb`.

Review output:

```
1. [Block] Cell 7 reports `retention = 25.4%` but the formula uses `event_date <= date_sub('2025-01-15', 6)` — that's a 6-day window, not 7. Re-derive D7 per docs/metric-definitions.md.

2. [Block] Cell 11 compares iOS vs Android with no CI. The reported delta is −0.8 pts; with n=12K each, the bootstrap CI likely overlaps zero. Add CIs.

3. [Major] No employee exclusion filter on the dim_customers join (cell 4). Internal employees are typically higher-engagement than customers; include whatever exclusion column your team uses (e.g., `is_employee = FALSE`) to avoid inflating retention numbers.

4. [Major] No mention of the BFCM cohort-tail effect (docs/business-info.md). The comparison window (Jan 1-7 vs Dec 25-31) straddles BFCM-driven cohorts; comparison may be apples-to-oranges. Acknowledge or restrict the window.

5. [Minor] Random seed not set in cell 2 setup. The bootstrap in cell 14 is nondeterministic between runs.

6. [Minor] Charts in cells 12-14 use default matplotlib palette; team palette is in skills/plotting-graphs/SKILL.md (Lavender Earth).

7. [Nit] Cell 1 has commented-out experimental code; remove before commit.

Verdict: NEEDS REVISION. Items 1-2 must be fixed before sharing the result. 3-4 should be addressed; 5-7 cleanup.
```

## What to add to docs/ if missing

- A recurring review failure (e.g., team consistently misses `is_employee`) → add to `analyses/AGENTS.md` checklist.
- A template addition (e.g., "always include a `caveats` cell at the bottom") → propose update to `analyses/_template.md`.
