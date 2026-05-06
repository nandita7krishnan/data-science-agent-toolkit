# experiments/AGENTS.md

This folder is for **A/B test write-ups.** One folder per experiment.

The experiment registry (`docs/experiment-registry.md`) is the catalog. The detailed write-ups, code, and analyses live here.

Statistical rigor is the point of this folder. **No p-hacking, no peeking, no "directional" results dressed up as decisions.** When in doubt, the answer is "we don't know yet."

## Folder structure

```
experiments/
├── AGENTS.md
└── EXP-YYYY-NNN/                    (matches the registry ID)
    ├── README.md                    (the write-up — use the structure in the Post-launch section below)
    ├── proposal.md                  (the pre-launch design doc)
    ├── analysis.ipynb               (the read-out)
    ├── figures/                     (saved figures)
    └── data/                        (keep local if large; small reference fine)
```

## Pre-launch (proposal)

Every experiment has a `proposal.md` written *before launch* and reviewed by the Experimentation Council (per `docs/business-info.md`):

1. **Hypothesis.** What change, what expected effect, on what metric.
2. **Variants.** Control + treatment(s).
3. **Primary metric.** **One** primary metric. Decisions are made on it. Pre-register.
4. **Secondary / guardrail metrics.** What could go wrong (engagement drop, unsubscribe rate, latency).
5. **Sample-size / power calculation.** Minimum detectable effect (MDE), expected duration. **No experiment runs without this.**
6. **Stopping rules.** When can the experiment stop early? (Almost always: never, unless guardrails breach.)
7. **Segments to break out.** Pre-register these. Post-hoc segment-hunting is p-hacking.
8. **Analysis plan.** Which test (chi-squared / t-test / CUPED-adjusted), which adjustment for multiple comparisons.

## During the experiment

- **Don't peek.** Power calculations assume one analysis at the end. Multiple peeks at p-values inflate false positive rates. The Experiment Watcher dashboard shows assignment health (sample-ratio mismatch checks, etc.) — that's it.
- **Watch guardrails.** If a guardrail metric breaches, that *is* the early-stop trigger. Document and act.
- **No mid-experiment changes** to the variant or the metric definition.

## Post-launch (write-up)

The write-up is the final entry in `README.md`. Use the template structure (see `experiment-registry.md` format). Run:

1. `data-quality-checker` on `fct_experiment_assignments` and the metric source tables.
2. `metric-calculator` on the primary metric, per variant.
3. `notebook-reviewer` on the analysis notebook.
4. `analysis-summarizer` for the audience-formatted output.

Then add the row to `docs/experiment-registry.md` (see its format) and link to this folder.

## Statistical rigor checklist (the actual one)

- [ ] **Sample-ratio mismatch**: assignment counts within ±0.5% of designed split.
- [ ] **Pre-period equivalence**: control and treatment had similar pre-period metrics. If not, randomization may have failed.
- [ ] **Power achieved**: actual sample size reached the pre-registered minimum.
- [ ] **Primary metric**: effect size, **CI**, p-value (or Bayesian posterior). State the test used.
- [ ] **Practical significance**: is the effect size large enough to matter to the business, even if statistically significant?
- [ ] **Guardrails**: each one, with effect size and CI. Any breaches called out explicitly.
- [ ] **Segments** (only the pre-registered ones): effect size + CI per segment, with multiple-comparisons correction (Bonferroni or BH).
- [ ] **Heterogeneity**: did the effect differ across segments? Document, don't quietly average.
- [ ] **Novelty / primacy effects**: for long-running experiments, plot the metric over time within the experiment window — early-window effects can fade.
- [ ] **Decision and rationale**: ship / don't ship / inconclusive. With reasons grounded in the numbers.
- [ ] **Learning**: a one-paragraph "what we now believe that we didn't before" — even if the experiment "failed." Goes into the registry.

## Hand-off

- **To registry**: copy the structured fields into `docs/experiment-registry.md`.
- **To leadership / stakeholders**: `analysis-summarizer` with audience = `exec` or `PM`.
- **To the team**: post in `#experimentation` with the write-up link.
