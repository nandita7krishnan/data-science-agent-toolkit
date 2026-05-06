# Experiment Registry

Past A/B tests. **Check here before re-running a similar experiment.** Failed experiments are as informative as successes — keep them.

> Forking? Empty the example entries below; keep the format. Backfill from your team's existing experiment tooling if you have one.

---

## Active experiments

Experiments currently running. Watch via the **Experiment Watcher** dashboard (`docs/dashboards.md`).

| Experiment ID | Hypothesis | Variant | Owner | Started | Expected end |
|---|---|---|---|---|---|
| `EXP-2025-014` | New checkout layout reduces cart abandonment | Control vs single-page checkout | DS — Conversion | 2025-01-10 | 2025-02-07 (planned) |
| `EXP-2025-015` | Personalized PDP reco lifts AOV by 3% | Control vs reco-block-v2 | DS — Personalization | 2025-01-12 | 2025-02-09 (planned) |

---

## Completed experiments — last 12 months

Reverse-chronological. Use this to answer "have we tried X before?"

### `EXP-2024-042` — Free-shipping threshold lowered to $50

- **Hypothesis:** Lowering the free-shipping threshold from $75 → $50 increases conversion rate by ≥ 5% with neutral GMV impact.
- **Window:** 2024-11-04 to 2024-12-02 (4 weeks)
- **Variants:** Control ($75) vs Treatment ($50)
- **Primary metric:** Session conversion rate
- **Result:** **Shipped (treatment).** CR +6.2% (95% CI [+3.4%, +9.0%], p<0.01). GMV per session −0.8% (CI overlapped zero, p=0.41). Net positive on AOV-adjusted revenue.
- **Caveats:** Window included the start of holiday season — re-evaluating in Q1 to confirm holds outside peak.
- **Decision:** Ship. Re-evaluate Q1 2025.
- **Learning:** AOV decrease was concentrated in the $50–$75 band as expected. Mix-shift toward smaller orders did not cannibalize larger orders.

### `EXP-2024-038` — Push notification "abandoned cart" timing

- **Hypothesis:** Sending the abandoned-cart push at 1 hour (vs current 4 hours) increases recovery by ≥ 10%.
- **Window:** 2024-09-15 to 2024-10-13 (4 weeks)
- **Variants:** Control (4h) vs Treatment (1h)
- **Primary metric:** Cart-recovery rate (purchase within 24h of abandonment)
- **Result:** **Did not ship.** Recovery rate −2.1% (CI [−4.5%, +0.3%], p=0.08), unsubscribe rate +18% (CI [+9%, +27%], p<0.01).
- **Caveats:** Effect concentrated in iOS users; Android trend was directionally positive but not significant.
- **Decision:** Don't ship at 1h. Test 2h variant in Q1 2025.
- **Learning:** Push fatigue is real and shows up in unsubscribe rate before it shows up in primary metric. Always include unsubscribe as a guardrail.

### `EXP-2024-029` — Email subject-line emoji

- **Hypothesis:** Adding a single emoji to promotional email subject lines lifts open rate by ≥ 3%.
- **Window:** 2024-07-08 to 2024-07-29 (3 weeks)
- **Variants:** Control (no emoji) vs Treatment (one contextual emoji)
- **Primary metric:** Open rate
- **Result:** **Inconclusive.** Open rate +1.2% (CI [−0.1%, +2.5%], p=0.07). Click rate flat. Unsubscribe flat.
- **Caveats:** Power was low — campaign volume during the window was 30% below plan due to a content-team backlog.
- **Decision:** Don't ship; rerun with full power.
- **Learning:** Always confirm planned campaign volume *before* the test starts. Underpowered tests are wasted weeks.

### `EXP-2024-021` — Logged-in homepage personalization

- **Hypothesis:** Personalized homepage hero (vs editorial hero) lifts session conversion by ≥ 5% for logged-in repeat buyers.
- **Window:** 2024-05-06 to 2024-06-03 (4 weeks)
- **Variants:** Control vs personalized
- **Primary metric:** Session conversion rate (logged-in repeat buyers)
- **Result:** **Shipped.** CR +7.8% (CI [+4.6%, +11.0%], p<0.001). AOV neutral.
- **Caveats:** Test population excluded logged-out and first-time buyers; result does not generalize.
- **Decision:** Ship for logged-in repeat buyers. Plan follow-up for first-time buyers.
- **Learning:** Personalization works when there's enough behavioral signal. Don't extrapolate to cold-start users.

---

## Format

Every entry uses this structure. Add new entries above the oldest, reverse-chronological.

```markdown
### `EXP-YYYY-NNN` — Short descriptive name

- **Hypothesis:** What you expected to happen, with effect size threshold.
- **Window:** Start date to end date.
- **Variants:** Control vs treatment(s).
- **Primary metric:** The single metric the decision rests on.
- **Result:** **Shipped / Did not ship / Inconclusive.** Effect size, CI, p-value.
- **Caveats:** Power, contamination, novelty effects, segment heterogeneity, anything that affects interpretation.
- **Decision:** What was decided and why.
- **Learning:** What the team takes forward, even if the test "failed."
```

## Rules

1. **Log every experiment**, even the ones you didn't ship. The "we tried X and it didn't work" entries save the team from re-running them.
2. **Power calculations belong in the proposal**, not here. This registry is the *outcome*. Detailed proposals live in `experiments/<exp-id>/`.
3. **Don't reuse experiment IDs.** Once `EXP-2024-042` is logged, that ID is dead.
4. **Update on decision change.** If a shipped experiment is later rolled back (e.g., long-term effect didn't hold), add a `**[ROLLED BACK YYYY-MM-DD]**` annotation with the reason.

## Common questions this should answer

- "Has anyone tested free shipping at $X before?" → search this file.
- "What did we learn about push notification timing?" → see EXP-2024-038.
- "Does personalization work for first-time buyers?" → it hasn't been tested (EXP-2024-021 was repeat-buyer-only) — opportunity.
