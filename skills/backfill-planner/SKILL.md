---
name: backfill-planner
description: Plan a safe backfill — scope, batching, idempotency precheck, cost estimate, monitoring, rollback path — output a runbook before any execution.
when_to_invoke:
  - "Plan a backfill for X"
  - "We need to rerun job Y for the last 30 days"
  - "Backfill table Z from <start> to <end>"
  - Before any backfill runs.
---

# Backfill Planner

Backfills are the most common cause of "we paid $40K to recompute a metric that was already right." Plan before executing. **The plan is the deliverable** — actually running the backfill is a separate step that requires explicit approval.

## Inputs

- Job / table to backfill.
- Date range (`start_date`, `end_date`).
- Reason (recovering from upstream issue, model rewrite, schema change, etc.).

## Outputs

A backfill runbook with:

1. **Scope** — exact partitions, tables, downstream consumers affected.
2. **Idempotency precheck** — verification the job is safe to rerun.
3. **Batch strategy** — how to chunk the work.
4. **Cost estimate** — scan size + compute cost.
5. **Monitoring** — what alerts to watch and silence.
6. **Rollback path** — how to undo if it goes wrong.
7. **Approval block** — explicit sign-off needed before execution.

## Procedure

1. **Read `docs/pipelines.md`** for the job: schedule, dependencies, watermark, owner.
2. **Scope:**
   - Which exact partitions? (e.g., `event_date BETWEEN '2024-12-01' AND '2024-12-15'`)
   - Which downstream tables depend on this output? Trace via `docs/pipelines.md` dependency graph and code search.
   - Any consumers (dashboards, models, reports) that snapshot from these tables and might need refreshing too?
3. **Idempotency precheck:**
   - Is the job idempotent? Read the job's incremental strategy — `merge` and `insert_overwrite` are idempotent; raw `append` typically isn't (will produce duplicates).
   - If not idempotent, **stop**. Either modify the job to be idempotent first, or use a different recovery path (manually delete + reload).
   - If idempotent, do a dry-run on **one** historical partition first and verify the output is byte-identical (or within expected drift) to what's already there.
4. **Batch strategy:**
   - Days vs weeks per batch: target ~1-2 hour runtime per batch. If a single day takes 30 minutes, batch by week. If a day takes 4 hours, batch by half-day.
   - Sequential vs parallel: parallel is faster but contends for warehouse capacity and amplifies upstream load. Default sequential unless ≥ 30-day backfill.
   - Off-peak scheduling: backfills > 4 hours total should run during low-traffic windows.
5. **Cost estimate:**
   - Source scan size = (rows per day) × (days in range) × (avg row width). Look this up in `docs/schema/<table>.md`.
   - Compute cost = scan size × your warehouse's $/TB-scanned. Round generously.
   - Add a 30% buffer for retries.
6. **Monitoring:**
   - Which alerts will fire on rerun (row-count anomaly, freshness)? Pre-silence them for the backfill window.
   - Which alerts must keep firing (genuine data quality issues)? Don't silence those.
   - Who watches? On-call DE, plus the requestor.
7. **Rollback path:**
   - If the backfill produces wrong data, how do you revert? Common options:
     - Delta time-travel: `RESTORE TABLE ... TO VERSION AS OF <pre-backfill version>`.
     - Snapshot copy: `CREATE TABLE <target>_pre_backfill DEEP CLONE <target>` before starting, then point reads back if rollback needed.
     - For non-Delta: depends on the table format. Document the path.
   - If no clean rollback exists, the backfill is high-risk — need explicit approval from the table owner.
8. **Approval block:** the runbook ends with named sign-offs needed (table owner, downstream consumer team if scope is wide).
9. **Stop.** Don't execute. Hand the runbook to the user / table owner for approval.

## Big-data / safety guards

- **No backfills > 30 days** without notification in `#data-eng-announce` (per `docs/pipelines.md`).
- **No backfills of `fct_page_events`** for partitions older than the watermark (6h) without explicit owner approval — late-arriving rows have already been routed to `_late`.
- **No SCD2 backfills** without snapshotting `dim_customers` first — the SCD2 job rewrites `valid_from`/`valid_to`.
- **No `--full-refresh` of a multi-TB table** without explicit cost approval.

## Worked example

> User: "Backfill `int_sessions_stitched` for `2025-01-08` through `2025-01-14` because we fixed a session-stitching bug yesterday."

Runbook:

```
BACKFILL: int_sessions_stitched
Window: 2025-01-08 → 2025-01-14 (7 days)
Reason: session-stitching bug fixed in commit a7c91f4 (2025-01-15); historical sessions need recompute

Scope:
  - Direct: int_sessions_stitched, partitions session_date 2025-01-08..2025-01-14
  - Downstream depending on these partitions:
      - fct_attribution (rebuilds trailing 90 days daily; will pick up the change automatically tomorrow)
      - reporting/dashboards: "Funnel Health" — uses int_sessions_stitched joined with fct_orders
      - models/repeat-purchase-classifier — features include session counts; retrain after backfill if used downstream
  - Consumers to notify: #marketing-analytics (uses Funnel Health daily)

Idempotency precheck:
  - Strategy: merge, unique_key = session_id. Idempotent. ✓
  - Dry-run on session_date = 2025-01-07 (control day, pre-bug-fix is correct on that date):
      Run job in dev schema → diff vs prod. Diff should be empty. ✓ (verified)

Batch strategy:
  - 7 daily batches, sequential.
  - Each batch: estimated 12 min runtime on prod warehouse.
  - Total wall time: ~90 min.
  - Off-peak window: start 22:00 UTC.

Cost estimate:
  - Scan ~140 GB per day × 7 = ~1 TB total.
  - At $5/TB-scanned: ~$5; with 30% buffer ~$7.

Monitoring:
  - Pre-silence: int_sessions_stitched row-count anomaly alert (will trigger from the recompute delta).
  - Keep active: schema-drift alert, completion-time alert.
  - Watcher: @<dataset-owner> + on-call @data-eng-clickstream.

Rollback:
  - Delta time-travel snapshot taken before start: VERSION AS OF current.
  - On failure: `RESTORE TABLE int_sessions_stitched TO VERSION AS OF <captured_version>`.

Approvals needed before execution:
  - [ ] @int_sessions_stitched owner (clickstream eng)
  - [ ] @marketing-analytics (downstream consumer, since their dashboard will show different numbers post-backfill)
  - [ ] DS Manager (if comparing to prior reports — caveat post-backfill numbers)
```

## What to add to docs/ if missing

- A non-idempotent job that needed a workaround → flag this in `docs/pipelines.md` and propose making the job idempotent.
- A new pre-silence alert pattern that worked → add as a step in this skill's "Monitoring" procedure.
