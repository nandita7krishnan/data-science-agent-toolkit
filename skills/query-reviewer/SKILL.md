---
name: query-reviewer
description: Pre-execution SQL review — partition filters, SELECT *, cross joins, scan estimates, deterministic ordering, type coercion — surface issues before the query runs.
when_to_invoke:
  - "Review this SQL before I run it"
  - "Is this query safe?"
  - Always before running any query against a table > 1M rows.
  - The user is about to run a query they wrote and asked for a check.
---

# Query Reviewer

Catch the costly, the wrong, and the slow **before execution.** This is the guardrail skill — invoke it as the second-to-last step before running any non-trivial SQL.

## Inputs

- The SQL query (Spark SQL, ANSI-leaning).
- Optional: target table doc path (`docs/schema/<table>.md`) for cross-checking.

## Outputs

A pass/fail review with:

- **Cost estimate** — rough scan size, based on the partition filter (or absence of one).
- **Issues** — categorized as **Block** (must fix), **Warn** (likely problem), **Nit** (style).
- **Suggested rewrite** — if any **Block** issues, an updated query.

## Procedure

1. **Identify all source tables.** For each, read its `docs/schema/<table>.md` if available.
2. **Check partition filters.** For every partitioned table referenced, verify a filter exists on the documented partition column. **Missing or vague filter (`>= '2020-01-01'`) → Block.**
3. **Check `SELECT *`.** If present on a wide table, **Warn** — request specific columns. **Block** if the table is > 1B rows.
4. **Check joins:**
   - Implicit Cartesian (`FROM a, b WHERE`) → **Block.**
   - `CROSS JOIN` without comment → **Warn.**
   - Missing `is_current = TRUE` on SCD2 dimensions (e.g., `dim_customers`) → **Block.**
5. **Check determinism:**
   - `FIRST`/`LAST`/`COLLECT_LIST` without `ORDER BY` → **Warn.**
   - Window functions with ties in `ORDER BY` and no tie-breaker → **Warn.**
6. **Check filters:**
   - String comparison against an unquoted date or untyped literal → **Warn.** Suggest `DATE 'YYYY-MM-DD'`.
   - `= NULL` (always false) → **Block.** Use `IS NULL`.
7. **Check exclusions** per `docs/business-info.md` and `docs/metric-definitions.md`:
   - Querying an orders/revenue table without the canonical status filter (e.g., `status = 'completed'`) → **Warn.**
   - Querying an events table without the synthetic/bot exclusion column your team uses → **Warn.**
   - Querying any user-level metric without your team's standard test/employee exclusions (check `docs/data-classification.md` for column names) → **Warn.**
8. **Estimate scan.** Multiply partition coverage × table size from `docs/schema/<table>.md`. Call out anything > 1 TB.
9. **Suggest rewrite** if any **Block** issues exist. Re-run review on the rewrite (max once — don't loop).

## Big-data / safety guards

- **Block** any query estimated to scan > 5 TB. Require explicit user confirmation to proceed.
- **Block** any `DROP`, `DELETE`, `TRUNCATE`, `ALTER ... DROP` on production catalogs without explicit user confirmation.
- **Never auto-run** the rewritten query — return it for the user to inspect.

## Worked example

> User: "Review this query: `SELECT * FROM analytics_prod.core.fct_page_events WHERE customer_id = 'cus_abc'`"

Review:

- **Block** — no `event_date` filter on a partitioned table. Estimated scan: ~120 TB (full table). The query will time out.
- **Block** — `SELECT *` on a 30+ column event table. Pull specific columns.
- **Warn** — no `is_synthetic = FALSE` filter; will include load-test events.

Suggested rewrite:

```sql
SELECT
    event_id, event_ts, event_name, session_id, surface, page_path
FROM analytics_prod.core.fct_page_events
WHERE event_date >= date_sub(current_date(), 7)  -- pick the smallest range that answers your question
  AND customer_id = 'cus_abc'
  AND is_synthetic = FALSE
ORDER BY event_ts;
```

Re-review: passes. Estimated scan ~ 0.4 TB (7 days × 1 customer's worth of partition pruning, dominated by the daily partitions read).

## What to add to docs/ if missing

- A repeated cost mistake (same table, same forgotten filter) → reinforce the warning in `docs/schema/<table>.md` "Partitioning & filters" section.
- A repeated style nit (e.g., team adopts trailing comma) → propose update to `docs/style-guide.md`.
