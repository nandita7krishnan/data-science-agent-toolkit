# data-science-agent-toolkit

> A blueprint for building an **agentic harness** for your data science team — a structured context layer that turns any AI coding assistant (Cursor, Claude Code, Copilot, etc.) into a competent data science teammate rather than a confident guesser.

## What this is

AI agents don't know your metrics, your tables, or your gotchas, so they invent a DAU formula that's almost-right, miss the partition filter on a 120 TB table, and confuse GMV with revenue. This kit gives them that context so you stop correcting the same things repeatedly.<img width="718" height="239" alt="Screenshot 2026-05-06 at 3 01 18 PM" src="https://github.com/user-attachments/assets/17d24e13-4855-4b92-9639-62bff3399e5f" />


It solves three problems:

| Problem | Solution |
|---|---|
| Agent doesn't know your metrics, tables, or gotchas | **`docs/`** — seeded institutional memory the agent reads on-demand |
| Agent doesn't know which rules apply to its current task | **`AGENTS.md` hierarchy** — scoped instructions per folder, not one giant prompt |
| Agent doesn't know *how* to do data work correctly | **`skills/`** — 18 reusable playbooks for recurring data tasks |

Everything is plain markdown. Tool-agnostic. No config files, no SDKs.

The glue that ties it together is the **[Prompt Library](#prompt-library)** — 7 ready-to-paste prompts that drive the whole workflow: bootstrapping your docs from scratch, documenting a table, extracting a canonical metric definition, creating a new skill, locking in corrections so they don't repeat, running a monthly audit, and planning an analysis before a single query runs. Fork the repo, run Prompt 1, and you're moving.

## How to build your harness — three phases

Getting started is a three-phase process. Don't try to do it all at once.

### Phase 1 — Seed your documentation

The `docs/` folder is the agent's institutional memory. This is the highest-leverage thing you can do. A well-seeded `docs/` eliminates the corrections you'd otherwise make 10 times a day.

**Start here: run Prompt 1** (see [Prompt Library](#prompt-library) below). The agent will interview you and draft each doc from your answers.

| Doc file | What goes here | Fill this when |
|---|---|---|
| `docs/glossary.md` | Product codes, acronyms, team jargon | Day 1 — agents hallucinate these |
| `docs/metric-definitions.md` | Canonical SQL formulas for every metric you report (DAU, retention, GMV, AOV, churn, LTV, conversion…) | Day 1 — metric disagreements are the #1 agent failure mode |
| `docs/business-info.md` | Business model, customer segments, calendar effects (BFCM, Prime Day adjacency, etc.) | Day 1 |
| `docs/pipelines.md` | Scheduled jobs: name, schedule, SLA, owner, failure runbook | When you first ask the agent to debug a pipeline |
| `docs/data-owners.md` | Who owns what domain, escalation paths, on-call contacts | When you first need to escalate |
| `docs/data-classification.md` | PII columns, Confidential columns, sharing rules by destination | Before the agent touches customer data |
| `docs/style-guide.md` | SQL/Python conventions: casing, aliasing, formatting | Before the agent writes any production code |
| `docs/environment.md` | Python version, seeds, fixtures, reproducibility contract | Before the agent runs any stochastic step |
| `docs/dashboards.md` | Catalog of existing dashboards (check before building new) | After your first "can you build me a dashboard" request |
| `docs/experiment-registry.md` | Past A/B tests: hypothesis, dates, result, learning | Before the agent designs an experiment |
| `docs/schema/<table>.md` | Table-by-table grain, columns, partitions, gotchas | Run Prompt 3 for each table before querying it |
| `docs/external-tables/<table>.md` | Tables owned by other teams: SLA, owner, access, caveats | When you first consume a cross-team table |

> **Convention:** Anything in `docs/schema/` uses `_template.md` as the starting point. Run Prompt 3 to populate it semi-automatically.

### Phase 2 — Set up the AGENTS.md hierarchy

`AGENTS.md` files are what the agent reads *first* when working in a folder. They're short (~30 lines each), scoped to that folder's concerns, and the root one sets repo-wide conventions.

The hierarchy ships pre-built. You need to **customize it** for your stack:
<img width="718" height="239" alt="Screenshot 2026-05-06 at 3 01 40 PM" src="https://github.com/user-attachments/assets/76b719ab-a772-4c34-bf6f-635e46d57a10" />


```
AGENTS.md                  ← root: tech stack, conventions, guardrails (EDIT THIS FIRST)
├── docs/AGENTS.md         ← how to read/maintain context docs
│   ├── docs/schema/AGENTS.md          ← schema doc rules, grain, partitions
│   └── docs/external-tables/AGENTS.md ← external-team rules: SLA, access
├── skills/AGENTS.md       ← skill structure, when to invoke
├── analyses/AGENTS.md     ← notebook conventions, sampling rules
├── pipelines/AGENTS.md    ← idempotency, SLAs, partition contracts
├── reporting/AGENTS.md    ← dashboard/report conventions
├── experiments/AGENTS.md  ← A/B writeup template, stat rigor
└── models/AGENTS.md       ← training conventions, seeds, artifacts
```

**What to customize in each:**

- **Root `AGENTS.md`** — change the tech stack section (Python version, SQL dialect, notebook environment). This is your single biggest lever.
- **`models/AGENTS.md`** — add your modeling framework (sklearn, XGBoost, PyTorch), experiment tracking tool (MLflow, W&B), artifact storage path, and model registry.
- **`pipelines/AGENTS.md`** — add your orchestrator (Airflow, Prefect, Dagster), SLA thresholds, alerting channels.
- **`experiments/AGENTS.md`** — add your stats library, minimum detectable effect conventions, decision framework.
- **Per-folder `AGENTS.md`** — any rule that's *only* true for that folder lives here, not in root.

The agent reads the **closest** `AGENTS.md` to where it's working. This means it gets model-training rules when editing a model, not when debugging a pipeline.

### Phase 3 — Add skills

Skills are reusable playbooks — numbered procedures the agent follows for recurring tasks. **18 are included.** Download them, use as-is, or adapt to your tables.

**To create a new skill:** run Prompt 2 below, or use the `create-skill` command in Cursor.

#### Skills index

<img width="709" height="336" alt="Screenshot 2026-05-06 at 2 59 54 PM" src="https://github.com/user-attachments/assets/4b500133-a7ff-4642-9249-00863a679793" />


| Skill | What it does |
|---|---|
| **Analysis & Insight** | |
| `analysis-planner` | Turn a fuzzy stakeholder question into a structured plan — clarifying questions, hypotheses, metrics, data sources, method, and deliverable shape — before any query runs |
| `eda-profiler` | Profile any table — row counts, nulls, distributions, ranges, anomalies — sample-aware for big data |
| `metric-calculator` | Calculate a metric using the team's exact canonical definition from `docs/metric-definitions.md` — never improvises |
| `cohort-analyzer` | Build a cohort retention or conversion analysis — define the cohort, time windows, and produce the retention triangle |
| `funnel-analyzer` | Build a step-by-step conversion funnel — drop-off attribution, time-to-convert, segmented breakouts — from event data |
| `root-cause-investigator` | When a metric moves unexpectedly, decompose it by dimension, cohort, change-point to localize the cause |
| `analysis-summarizer` | Turn notebook output into a structured summary tailored to the audience — exec, Slack, email, or slide bullets |
| `query-reviewer` | Pre-execution SQL review — partition filters, `SELECT *`, cross joins, scan estimates — surface issues before the query runs |
| **Visualization** | |
| `plotting-graphs` | Produce publication-quality charts — chart-type selection, visual hierarchy, color, annotation, and the team's palette |
| **ML / Modeling** | |
| `feature-engineer` | Build a feature pipeline for tabular ML — encode, scale, impute, date-feature, assemble with `ColumnTransformer`, and run a leakage check |
| `model-evaluator` | Score a trained model honestly — right metric, naive baseline, bootstrap CIs, residuals/calibration, slice breakout |
| **Documentation & Quality** | |
| `schema-documenter` | Scan code + `DESCRIBE TABLE` → draft a `docs/schema/<table>.md` from the template, flagging gaps that need human input |
| `data-quality-checker` | Consumer-side ad-hoc data audit — duplicates, nulls, schema drift, freshness, row-count sanity — before trusting a table |
| `notebook-reviewer` | Fresh-eyes review — hidden assumptions, missing CIs, sample-size issues, p-hacking risk, missing seed, cell ordering |
| **Governance** | |
| `pii-scrubber` | Detect and mask PII in query output, logs, or notebook cells before sharing — applies the right transformation per data class |
| **Data Engineering** | |
| `pipeline-debugger` | Investigate a failed scheduled job — pull run status + error trace, classify transient vs code bug, check recent commits |
| `incremental-model-builder` | Write or modify an incremental SQL/dbt model — pick the right strategy, configure `unique_key`, handle late-arriving data |
| `backfill-planner` | Plan a safe backfill — scope, batching, idempotency precheck, cost estimate, monitoring, rollback path |

> `skills/_template/SKILL.md` is the starting point for any new skill you write.

---

## Quickstart (5 minutes)

1. **Fork** this repo. Rename to your team's repo.
2. **Open it** in Cursor (or Claude Code, Copilot, etc.). The agent discovers `AGENTS.md` automatically.
3. **Read `examples/analysis-walkthrough.md`** to see the full loop end-to-end.
4. **Run Prompt 1** below to start seeding your docs. Start with `glossary.md` and `metric-definitions.md` — these have the highest return.
5. **Customize root `AGENTS.md`** — change the tech stack section to match your Python version, SQL dialect, and notebook environment.
6. **Pick a real task.** Watch where the agent stumbles. Each stumble is a doc gap; close it with Prompt 5.
<img width="651" height="301" alt="Screenshot 2026-05-06 at 2 58 58 PM" src="https://github.com/user-attachments/assets/8fa675a1-25f9-4037-bf27-18cafc102589" />


---

## Repo tour

```
data-science-agent-toolkit/
├── README.md                  ← you are here
├── AGENTS.md                  ← root: tech stack, conventions, guardrails
├── CLAUDE.md                  ← @AGENTS.md (one line, for Claude Code)
├── .cursorrules               ← @AGENTS.md (one line, for Cursor)
├── .env.example               ← env var scaffolding; copy to .env, never commit .env
├── .gitignore                 ← pre-configured for Python / Jupyter / uv
│
├── docs/                      ← the team's institutional memory
│   ├── AGENTS.md              ← rules for maintaining this folder
│   ├── glossary.md            ← product codes, acronyms, jargon
│   ├── metric-definitions.md  ← canonical SQL formulas (DAU, retention, GMV, AOV, churn, LTV…)
│   ├── business-info.md       ← business model, customer segments, calendar effects
│   ├── pipelines.md           ← scheduled jobs: schedule, SLA, owner, runbooks
│   ├── style-guide.md         ← SQL/Python style: casing, aliasing, layout
│   ├── data-owners.md         ← who owns what, escalation paths, RACI
│   ├── environment.md         ← Python/Spark versions, seeds, fixtures, reproducibility
│   ├── data-classification.md ← PII / Confidential / Internal / Public
│   ├── dashboards.md          ← catalog of existing dashboards (check before building new)
│   ├── experiment-registry.md ← past A/B tests with hypotheses, results, learnings
│   ├── schema/                ← table-by-table docs
│   │   ├── AGENTS.md          ← schema doc rules, grain, partitions
│   │   ├── _template.md       ← copy this for every new table
│   │   ├── fct_orders.md
│   │   ├── fct_page_events.md
│   │   └── dim_customers.md
│   └── external-tables/       ← cross-team tables
│       ├── AGENTS.md          ← external-team rules: SLA, access
│       ├── _template.md
│       └── stg_marketing__campaigns.md
│
├── skills/                    ← 18 reusable playbooks
│   ├── AGENTS.md              ← skill structure, when to invoke, how to write one
│   ├── _template/SKILL.md     ← starting point for new skills
│   └── <skill-name>/SKILL.md  ← one per skill (see Skills index above)
│
├── analyses/                  ← ad-hoc / exploratory work (notebooks, deep dives)
├── pipelines/                 ← scheduled jobs with SLAs
├── reporting/                 ← recurring dashboards & reports
├── experiments/               ← A/B test write-ups
├── models/                    ← trained ML models + training code
├── data/                      ← local data (fixtures committed; raw excluded via .gitignore)
└── examples/
    └── analysis-walkthrough.md  ← end-to-end worked example
```

---

## Customizing for your team

The repo ships with **ABC Business** (a fictional online retailer) as example content throughout. To make it yours:

| Change | What to edit |
|---|---|
| **Replace ABC Business glossary** | `docs/glossary.md` — your product codes, acronyms, jargon |
| **Replace metric definitions** | `docs/metric-definitions.md` — your DAU, retention, revenue formulas |
| **Replace schema docs** | Run Prompt 3 for each of your real tables; delete ABC Business examples once replaced |
| **Replace pipelines** | `docs/pipelines.md` — your jobs, schedules, owners |
| **Replace owners** | `docs/data-owners.md` — your team contacts, escalation paths |
| **Swap SQL dialect** | Search for "Spark SQL" in `AGENTS.md` and `docs/style-guide.md`; adjust function names for Snowflake / BigQuery / DuckDB |
| **Rename functional folders** | If your team uses `dbt/`, `looker/`, etc., rename `pipelines/` and `reporting/` and update cross-references |
| **Adapt skills** | Skills are tool-agnostic; tweak the example queries in each `SKILL.md` to use your real tables |
| **Add model-specific conventions** | `models/AGENTS.md` — your framework, experiment tracker, artifact storage, registry |

---

## Prompt library

Seven prompts. Paste any into your AI coding assistant when the trigger fits.

### Prompt 1 — Bootstrap docs

> Use when: setting up the kit for the first time, or after a major reorg.

```
You are setting up a data-science context repo for our team. 

I want you to draft (or replace existing ABC Business placeholder content with) the following, based on a conversation with me:

1. Root AGENTS.md — adjust the tech stack section to match: <Python version, package manager, SQL dialect, notebook environment>.
2. docs/glossary.md — ask me for our product codes, acronyms, and team-specific jargon.
3. docs/metric-definitions.md — ask me for DAU, retention, revenue, conversion (whatever metrics we report).
4. docs/business-info.md — business model, customer segments, calendar periods that distort metrics.
5. docs/pipelines.md — top 5-10 scheduled jobs we depend on (name, schedule, owner, SLA, common failures).
6. docs/data-owners.md — who owns what, escalation paths.
7. docs/data-classification.md — our PII columns, our Confidential columns, sharing rules.
8. docs/dashboards.md — top 5-10 dashboards we already have.
9. docs/environment.md — confirm versions, seed pattern, time zone conventions.
10. docs/experiment-registry.md — last 4-8 completed experiments (we'll fill more in over time).

Per-folder AGENTS.md (analyses/, pipelines/, reporting/, experiments/, models/) — adapt the existing ABC Business examples to our reality.

Ask me clarifying questions, one batch at a time. Don't write any docs until I've answered the relevant questions for that doc. After each doc, show me the draft and ask for changes before moving on.

Don't write schema docs yet — those come from Prompt 3, one table at a time.
```

### Prompt 2 — Create a new skill

> Use when: you've asked the agent to do "this kind of thing" 3+ times. Pattern is real; codify it.

```
I keep asking you to <describe the recurring task — e.g., "compare a metric's distribution across two windows and tell me if the difference is plausible">. Read skills/_template/SKILL.md and at least 2 existing skills (good candidates: <names of similar skills>) for the level of detail expected.

Then draft skills/<kebab-case-name>/SKILL.md for this task. Include:
- Frontmatter (name, description, when_to_invoke triggers — at least 3)
- Inputs and Outputs sections
- A numbered Procedure
- Big-data / safety guards
- A worked example using our actual tables (per docs/schema/)
- A "what to add to docs/ if missing" section

Keep it terse — target 60-100 lines. After the draft, propose updates to skills/AGENTS.md if this skill changes the lane summary table.

Stop after the draft. I'll review before committing.
```

### Prompt 3 — Document one schema

> Use when: an undocumented table comes up in an analysis. Run this *before* querying it for real.

```
Run the schema-documenter skill (skills/schema-documenter/SKILL.md) on the table <full.table.path>:

1. Read docs/schema/_template.md and docs/schema/fct_orders.md (as a reference for the level of detail).
2. Run DESCRIBE EXTENDED on the table.
3. Grep the repo for usages (rg "<table_name>" --type sql --type py).
4. Cross-reference each column against docs/data-classification.md to mark PII / Confidential.
5. Look up the producing job in docs/pipelines.md.
6. Draft docs/schema/<table>.md (or docs/external-tables/<table>.md if the producer is an external team).
7. Mark every gap that requires human knowledge with **[HUMAN: confirm]** — especially: grain interpretation, business gotchas, owner contact.

Output the draft as a single file. List the [HUMAN: confirm] items at the bottom so I can answer them.
```

### Prompt 4 — Extract a metric definition

> Use when: a metric is calculated in multiple places with disagreeing definitions, or it isn't documented yet.

```
Find every place we calculate <metric name> in this repo: scan analyses/, pipelines/, reporting/, models/, and any docs/ files.

For each occurrence:
- Quote the exact formula or SQL snippet.
- List the table(s) used.
- Note any exclusions or filters applied.
- Note the file it came from.

Then reconcile:
- What's the dominant pattern?
- Where do they disagree, and on what?
- Which version matches what the team actually wants? (Ask me if unclear.)

Propose a canonical formula for docs/metric-definitions.md, following the format of existing entries (definition, source table, formula in SQL, exclusions, gotchas). Include any places that disagree as candidates to update once the canonical version lands.

Don't update any analyses/pipelines yet — just propose the metric-definitions.md addition.
```

### Prompt 5 — Retroactive correction

> Use when: you've corrected the agent. Make the correction stick so it doesn't repeat.

```
I just corrected you about <thing — e.g., "GMV uses order_subtotal, not order_total">.

1. Identify the closest AGENTS.md or docs/ file where this correction belongs. Don't add it to a global place if a more specific home exists.
2. Propose the exact edit (before/after, or a clean diff). Include enough context to drop in.
3. If a new term emerged that we don't have in docs/glossary.md, add a one-line entry there too.
4. If you spot that this correction would have been caught by an existing skill (and the skill missed it), propose an update to that skill's procedure or guards.

Output the edits but don't commit yet — show me, then I'll commit.
```

### Prompt 6 — Audit existing docs

> Use when: monthly maintenance, or after a quarter of accumulating Prompt-5 corrections.

```
Audit our docs and skills:

1. Read every AGENTS.md (root + every subdirectory).
2. Read every file in docs/.
3. Read every skills/<name>/SKILL.md.

Produce an audit report with:

- **Contradictions** — where two docs say different things about the same fact.
- **Stale references** — links to files that don't exist, owners who left, deprecated tables/jobs still referenced.
- **Missing pieces** — patterns that should have a doc but don't (metric used in 3 analyses but not in metric-definitions.md; table queried but not in docs/schema/).
- **Skills that drift from their template** — frontmatter missing fields, no worked example, exceeds the length budget significantly.
- **Cross-doc redundancy** — facts duplicated where progressive disclosure would be cleaner.

Don't edit anything yet — just produce the audit. Group by severity (Block / Major / Minor / Nit). I'll triage and assign edits to follow-up Prompt-5 invocations.
```

### Prompt 7 — Plan an analysis

> Use when: a stakeholder pings you with a fuzzy ask. Run this before opening a notebook.

```
Stakeholder ask: <paste verbatim>.

Run the analysis-planner skill (skills/analysis-planner/SKILL.md):

1. Check docs/dashboards.md to see whether an existing dashboard already answers this.
2. Ask me clarifying questions (numbered, answerable): time window, audience, surfaces, regions, exclusions, deliverable shape.
3. State the prior expectation and what would surprise you.
4. List the metrics (from docs/metric-definitions.md) and tables (from docs/schema/) you'd use.
5. Pick the method (descriptive / cohort / funnel / root-cause / experiment / predictive) and justify in one sentence.
6. Sketch the deliverable shape and estimate effort.
7. Stop. Don't run any queries until I've signed off on the plan.
```

