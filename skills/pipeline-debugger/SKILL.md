---
name: pipeline-debugger
description: Investigate a failed scheduled job — pull run status + error trace, classify transient vs code bug, check recent commits, output remediation plan.
when_to_invoke:
  - "Job X failed — what happened?"
  - "Pipeline alert just fired"
  - "Investigate the failure of <job name>"
  - On-call paged with a job-failure ticket.
---

# Pipeline Debugger

A failed pipeline almost always falls into one of three buckets: **transient** (retry it), **code bug** (fix it), or **data quality** (call the upstream team). The job is to classify quickly and route correctly — don't reflexively rerun.

## Inputs

- Job name (must match an entry in `docs/pipelines.md`).
- Run ID or failure timestamp.
- Optional: linked alert ticket / Slack message.

## Outputs

- A structured incident note with: classification, evidence, remediation, and next-on-failure path.

## Procedure

1. **Read `docs/pipelines.md`** for the job. Note the owner, schedule, watermark, and listed common failures.
2. **Pull run status.** From the orchestrator (Airflow / Databricks Workflows / dbt Cloud / etc.): start time, end time, failed task, exit code.
3. **Pull the error trace** from the failed task. Read it — don't just paste it. Look for:
   - Connection / network errors → likely **transient**.
   - Source-replica lag, "table not found" → likely **transient** if temporary; **upstream issue** if persistent.
   - SQL parse errors, column-not-found, type mismatch → **code bug** (introduced by a recent change) or **schema drift** (upstream).
   - OOM / timeout → **infrastructure**, may be transient or a real bug.
4. **Check recent commits to the job's code:**

```bash
git log --since='3 days ago' -- <path-to-job>
```

If a commit landed near the failure timestamp and touched the failing area, that's the prime suspect.

5. **Check upstream sources.** For tables produced by other teams (per `docs/external-tables/`), confirm the upstream produced its expected partition.
6. **Classify and route:**
   - **Transient** (network blip, replica lag): retry once. If it fails again, escalate to **code bug / upstream**.
   - **Code bug**: identify the suspect commit, page the owner (per `docs/data-owners.md`), don't retry.
   - **Upstream issue**: notify the upstream team via their channel (per `docs/external-tables/<table>.md` bug-report path). Don't retry — it'll fail again.
   - **Infrastructure (OOM, timeout)**: increase resources via the orchestrator config; rerun once. If it fails, dig into query plan / explain.
7. **Document.** Add the failure to `docs/pipelines.md` "Common failures" if it's a new mode. Even one-off failures are worth a line.
8. **Output the incident note** in the standard format below.

## Big-data / safety guards

- **Don't reflexively rerun** a failed job. Reruns of upstream-issue jobs amplify the upstream problem and can fill log volumes.
- **Backfills** triggered as part of debugging must use `backfill-planner` skill — never just `rerun all of last week`.
- **Never edit a running job's code without notifying the owner.** Use a hotfix branch + page the owner.

## Worked example

> User: "Job `fct_page_events_load` failed at 03:15 UTC. The 02:00 hour partition is missing."

1. Read `docs/pipelines.md` → owner is Data Eng — Clickstream (`#data-eng-clickstream`). Common failures listed: schema-registry mismatch on Kafka.
2. Run status: failed at 03:15 UTC; failed task is `kafka_to_delta_02h`.
3. Error trace excerpt:

```
org.apache.spark.SparkException: Schema mismatch for topic your-team.web.events:
  expected field `purchase_currency` (STRING) — actual: missing
```

This is a Kafka schema-registry mismatch — listed as a common failure for this job.

4. Recent commits to the job: none in the last 3 days. Suggests upstream change.
5. Upstream check: the `web-events` producer team deployed yesterday. Likely they added a field but the Schema Registry wasn't updated.
6. **Classification: upstream issue** (Kafka Schema Registry not in sync).
7. **Action:**
   - Notify `#data-eng-clickstream` and `#web-eng` about the schema-registry mismatch.
   - **Do not rerun.** It will fail until the registry is updated.
   - Once the registry is fixed, run `backfill-planner` to plan rerunning the missing 02:00–04:00 partitions.

8. **Incident note:**

```
Incident: fct_page_events_load failed 2025-01-15 03:15 UTC
Failed task: kafka_to_delta_02h
Classification: upstream issue (Kafka Schema Registry mismatch on your-team.web.events)
Evidence: error trace shows missing `purchase_currency` field; web-eng deployed 2025-01-14 18:00 UTC
Action taken: notified #data-eng-clickstream and #web-eng; awaiting registry update
Action pending: rerun affected partitions (02:00-04:00) once registry is fixed; use backfill-planner
Doc updates: none — this matches the existing "Schema-registry mismatch" entry in docs/pipelines.md
```

## What to add to docs/ if missing

- A novel failure mode → add to `docs/pipelines.md` "Common failures" for the job.
- A repeating upstream issue → tag in `docs/external-tables/<source>.md` Migration / deprecation notes.
