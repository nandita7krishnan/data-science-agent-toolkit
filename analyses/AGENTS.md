# analyses/AGENTS.md

This folder is for **ad-hoc / exploratory work.** Notebooks, one-off scripts, deep dives. Things that have a beginning and an end, are not scheduled, and may or may not produce a deliverable.

Anything **scheduled** lives in `pipelines/`. Anything **recurring as a report** lives in `reporting/`. A/B test write-ups live in `experiments/`. Trained ML models + training code live in `models/`.

## Folder structure

```
analyses/
├── AGENTS.md                 (this file)
├── _template.md              (lightweight template — copy when starting a new analysis)
└── YYYY-MM/                  (group by month)
    └── <slug>/
        ├── README.md         (the question, the answer, the method, the caveats)
        ├── notebook.ipynb    (the work)
        └── figures/          (optional — small saved figures)
```

The analysis slug is short, descriptive, and kebab-case. Examples: `d7-retention-drop`, `bfcm-cohort-quality`, `paid-social-attribution-deep-dive`.

## When you start a new analysis

1. **Run the `analysis-planner` skill** before opening a notebook. The plan goes in `README.md`. Don't skip this — analyses without a plan turn into queries-without-questions.
2. **Copy `_template.md` to `analyses/YYYY-MM/<slug>/README.md`** and fill it in. The template is short on purpose.
3. **Create a notebook** with the standard structure (header / setup / load / analyze / summary). See "Notebook conventions" below.
4. **Set the seed** in the setup cell per `docs/environment.md`.
5. **Add the `figures/` subdir only if you save figures.** Empty dirs aren't tracked.

## Notebook conventions

- **Top markdown cell**: title, author, date, the question (verbatim).
- **Setup cell**: imports, env vars from `.env`, seed, display options.
- **Load cell**: the queries. Parameterized SQL only.
- **Sanity-check cell**: row counts, partition coverage, freshness of source tables (per `data-quality-checker`).
- **Analysis cells**: one logical step per cell. No multi-page mega-cells.
- **Summary cell**: bullet-point findings, with the headline first. Run `analysis-summarizer` against this cell.
- **Caveats cell**: explicit. Don't bury caveats in prose elsewhere.

## Reproducibility checklist (before claiming "done")

- [ ] `Restart & Run All` produces identical output.
- [ ] Random seed set (per `docs/environment.md`).
- [ ] All exclusions applied (`is_synthetic = FALSE`, `is_test_account = FALSE AND is_employee = FALSE`, `is_current = TRUE` on SCD2).
- [ ] Metrics use `docs/metric-definitions.md` formulas verbatim.
- [ ] No PII in committed output (`pii-scrubber` run).
- [ ] No raw `cost_of_goods_sold` / `gross_margin` / `lifetime_value` in any committed figure or summary.
- [ ] Findings reviewed by `notebook-reviewer` skill before sharing.
- [ ] `README.md` filled in with the headline.

## Sampling

- For exploration on tables > 10M rows in the slice, **sample first**:
  - `df.sample(fraction=0.01, seed=SEED)` (Spark)
  - `TABLESAMPLE (1 PERCENT)` (SQL)
- State the sample size and seed in the notebook.
- For final numbers, run on the full slice — but only after the analysis is right on the sample.

## Hand-offs

- **To stakeholders**: `analysis-summarizer` skill formats for the audience.
- **To the team for review**: `notebook-reviewer` skill produces review comments.
- **To `experiments/`**: if the analysis turned into an A/B-test write-up, move it.
- **To `pipelines/`**: if the analysis spawned a recurring need, productionize the query as a job.
