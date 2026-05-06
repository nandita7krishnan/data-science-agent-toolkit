# pipelines/AGENTS.md

This folder is for **scheduled jobs with SLAs** that this team owns. Jobs that produce or transform data on a recurring schedule.

Cross-team-owned pipelines are documented in `docs/external-tables/`, not here. Ad-hoc work lives in `analyses/`. Trained ML models with retraining pipelines live in `models/` (the model is the artifact; training is its pipeline).

## Folder structure

```
pipelines/
├── AGENTS.md           (this file)
└── <job-name>/
    ├── README.md       (purpose, schedule, SLA, owner — match the catalog in docs/pipelines.md)
    ├── job.py          (or .sql, or dbt model — the actual code)
    ├── tests/          (unit tests; integration tests in repo-level test runner)
    └── runbook.md      (failure modes, remediation, escalation)
```

## Conventions (hard rules)

1. **Idempotency.** Every scheduled job must be idempotent — running the same partition twice produces the same result. Verify with `incremental-model-builder` skill before declaring done.
2. **Partition contracts.** Every job declares its output partition schema (column, type, granularity) at the top of the README. Downstream consumers depend on this — breaking it is a contract break.
3. **SLA in `docs/pipelines.md`.** The catalog is canonical. The job's own README links to the catalog entry, doesn't duplicate it.
4. **Watermark handling.** Late-arriving rows beyond the watermark go to `<table>_late`. Don't backfill the original partition without `backfill-planner`.
5. **Ownership at the top.** Every job's README has the owning team and Slack channel in the first 5 lines. PagerDuty rotation linked.
6. **Runbook required.** Every job has a `runbook.md` with at least 3 failure modes and the remediation path for each. New failure modes get appended; never deleted.

## Adding a new job

1. **Open the discussion in `#data-eng-<domain>`** before you write the code. New jobs have downstream costs (compute, on-call burden).
2. **Catalog entry first.** Add to `docs/pipelines.md` *before* the job is wired up. Include schedule, SLA, watermark, owner, common failures (start with at least 3).
3. **Code goes here**, in `pipelines/<job-name>/`.
4. **Tests required:**
   - Unit: pure-function transforms.
   - Integration: a sample-data round-trip on a dev catalog.
   - Smoke test: scheduled "fail-fast" run that errors on schema drift.
5. **CI/CD** — the team's deploy pipeline applies. Document the deploy command in `runbook.md`.

## Modifying an existing job

- **Trivial change** (logic equivalence, comment): land normally.
- **Behavior change** (output column added, partition behavior change, business-logic change): **must** notify downstream consumers (per `docs/pipelines.md` dependency graph) and update `docs/schema/<output-table>.md`. Don't ship until consumers acknowledge.
- **Breaking change** (output column dropped, type changed): treat as a deprecation. Run new + old in parallel for at least one full SLA cycle. Migrate consumers. Then remove.

## Failure response

When a job fails:

1. Run `pipeline-debugger` skill.
2. Classify (transient / code / upstream / infrastructure).
3. Route per `docs/data-owners.md`.
4. Add the failure mode to `runbook.md` if it's new. Even one-offs are worth a line.

## "Should this be a pipeline?" checklist

Before promoting an ad-hoc analysis to a scheduled job:

- [ ] Will this output be consumed > 1× / week, indefinitely?
- [ ] Is the consumer a person (use `reporting/`) or a system (use `pipelines/`)?
- [ ] Does the team have on-call coverage for this domain? (If not, talk to DE Manager.)
- [ ] Is the output going into a `dim_*` / `fct_*` / `int_*` table? (If not, this might be `reporting/` instead.)
- [ ] Do you have a rough SLA in mind?

If you answered "no" to any, it's probably not a pipeline yet.
