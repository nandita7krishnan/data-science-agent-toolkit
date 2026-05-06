# AGENTS.md — Root

You are working in **`data-science-agent-toolkit`**, a fork-and-go context repo for data science teams using AI coding assistants. Your job is to act like a competent data science teammate, not a code-completion engine.

This file is the index. Read the closest `AGENTS.md` to where you're working (each subdirectory has its own with rules specific to that scope). When a `docs/<file>.md` is referenced below, treat it as canonical — do not improvise alternatives.

## Tech stack

- **Language:** Python 3.12, managed with `uv`
- **SQL dialect:** Spark SQL (ANSI-leaning — uses `current_date()`, `date_sub()`, etc., but avoids Delta-specific commands like `OPTIMIZE`/`ZORDER`).
  - **Forking?** Swap to your dialect: search for `Spark SQL` in this file and `docs/style-guide.md`, replace function names, and update `docs/schema/_template.md`.
- **Notebook environment:** Jupyter (assume `ipykernel` available). Notebooks live under `analyses/`.
- **Visualization:** matplotlib + plotly. Color palettes defined in `skills/plotting-graphs/SKILL.md`.

## How to run things

These are placeholders — replace with your team's actual commands.

```bash
uv sync                         # install deps
uv run pytest -m unit           # tests
uv run ruff check .             # lint
uv run jupyter lab              # notebooks
```

## Conventions (always)

1. **Never `SELECT *`** without an explicit partition filter on partitioned tables. Cost-of-mistake is real (see `docs/schema/`).
2. **Parameterize SQL.** No f-strings, no `.format()`, no string concatenation for query construction.
3. **Validate row counts after every join.** Print `before` and `after` counts. If they changed unexpectedly, stop and explain.
4. **Specify the grain** when describing or building any table: "one row per X per Y."
5. **Sample first.** On any table > 1M rows, run on a 0.1–1% sample before the full query.
6. **Set seeds** per `docs/environment.md` for any stochastic step (sampling, train/test split, model init).
7. **Ask clarifying questions before proceeding** when the request is ambiguous. Do not guess at metric definitions, time ranges, exclusions, or the audience for the deliverable.

## Progressive disclosure — when to read what

Read these on-demand. Do not pre-load.

| If you need to know... | Read |
|---|---|
| What does `dim_*`, `fct_*`, `stg_*` mean? | `docs/glossary.md` |
| How do we calculate DAU, retention, churn, LTV? | `docs/metric-definitions.md` |
| What's the business context for this product? | `docs/business-info.md` |
| Which jobs run on what schedule? | `docs/pipelines.md` |
| SQL/Python style: casing, aliasing, layout | `docs/style-guide.md` |
| Who owns this domain? Who do I escalate to? | `docs/data-owners.md` |
| Python version, seeds, fixtures, reproducibility | `docs/environment.md` |
| Is this column PII? Can I share the output? | `docs/data-classification.md` |
| Does a dashboard already answer this question? | `docs/dashboards.md` |
| Has someone tested this hypothesis before? | `docs/experiment-registry.md` |
| Schema, partitions, gotchas for `<table>` | `docs/schema/<table>.md` |
| Schema for an externally-owned table | `docs/external-tables/<table>.md` |
| How to do a specific kind of work | `skills/<skill-name>/SKILL.md` |

## Functional folders — read the local AGENTS.md *before* working there

| Folder | Purpose |
|---|---|
| `analyses/` | Ad-hoc / exploratory notebooks |
| `pipelines/` | Scheduled jobs with SLAs |
| `reporting/` | Recurring dashboards & reports |
| `experiments/` | A/B test write-ups |
| `models/` | Trained ML models + training code |

## Guardrails (hard rules)

- **No destructive queries** (`DROP`, `DELETE`, `TRUNCATE`, `ALTER ... DROP`) without explicit user confirmation, even on dev tables.
- **`EXPLAIN` before running** any query against a table > 1M rows or when partition coverage is unclear.
- **No committed credentials.** Never commit `.env`; use `.env.example` as the template.
- **Run `pii-scrubber` skill before pasting any query output, log line, or notebook cell that may contain PII.** "May contain" is broad — when in doubt, scrub.
- **Before claiming a task is done:** lint, run tests, verify no secrets staged, confirm the deliverable matches what was asked.

## Self-improvement protocol

When the user corrects you about a convention, a metric definition, a data nuance, or a workflow:

1. Identify the closest doc (`AGENTS.md`, `docs/<file>.md`, `docs/schema/<table>.md`) where this correction belongs.
2. Propose the **exact edit** (a diff or quoted before/after) so it sticks.
3. If a new term emerged, add a one-line note to `docs/glossary.md`.
4. If you spotted a recurring pattern (you keep being asked to do the same kind of thing), suggest writing a new skill — see `skills/AGENTS.md` and run Prompt 2 in `README.md`.

Corrections should compound, not repeat.
