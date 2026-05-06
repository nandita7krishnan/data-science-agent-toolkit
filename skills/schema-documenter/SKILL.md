---
name: schema-documenter
description: Meta-skill — scan code + DESCRIBE TABLE → draft a docs/schema/<table>.md from the template, flagging what only a human can know.
when_to_invoke:
  - "Document this table"
  - "Generate a schema doc for X"
  - "Create docs/schema/<table>.md"
  - The user asks about a table that has no schema doc yet.
---

# Schema Documenter

Generates a draft `docs/schema/<table>.md` for the human to review and finish. The agent can fill in everything *discoverable* from the database and codebase; the human fills in **business meaning, gotchas, and grain interpretation**.

## Inputs

- Full table path (`<catalog>.<schema>.<table>`).
- Optional: known partition column, expected grain.

## Outputs

- A draft `docs/schema/<table>.md` with the discoverable fields filled in.
- A clearly-marked `**[HUMAN: confirm]**` block for anything that requires business knowledge.

## Procedure

1. **Read `docs/schema/_template.md`** as the starting structure. Also read 1-2 existing schema docs (e.g., `fct_orders.md`) for the level of detail expected.
2. **Run `DESCRIBE EXTENDED <full_path>`** (or equivalent) and capture: column names, types, nullability, partition column(s), table location, last-modified.
3. **Run a quick row-count and grain-key check** to sanity-check the assumed grain:

```sql
SELECT COUNT(*) AS rows, COUNT(DISTINCT <suspected_grain_key>) AS distinct_keys
FROM <full_path>
WHERE <partition_column> = current_date();  -- adjust as appropriate
```

If `rows == distinct_keys`, grain is "one row per `<grain_key>`." If not, the grain involves a composite key — flag for the human.

4. **Grep the repo for usages** to surface conventions (`rg "<table_name>" --type sql --type py`). Note common filters, aliases, and join patterns.
5. **Check if the table is partitioned**. The `DESCRIBE EXTENDED` output reveals this. If yes, fill in "Partition column" and write the "always filter on partition" rule.
6. **Compute basic stats** for the column table: nullability, sample distinct count for low-cardinality columns. Don't compute distribution stats — that's the `eda-profiler` skill's job.
7. **Look up data classification.** Cross-reference each column against `docs/data-classification.md`. PII columns get the `**PII**` mark in the table.
8. **Look up the producing job.** Search `docs/pipelines.md` for the table name. If found, link it. If not, mark `**[HUMAN: confirm producing job]**`.
9. **Mark every gap** that requires human knowledge with `**[HUMAN: confirm]**`:
   - **Grain** ("Is this really one row per X?")
   - **Business meaning** of the table
   - **Gotchas** (only the team knows these)
   - **Owning team and Slack channel**
   - **SLA / freshness commitments**
10. **Write the draft** under `docs/schema/<table>.md` (or `docs/external-tables/<table>.md` if the producer is an external team).
11. **Hand to the user** with a list of the `[HUMAN: confirm]` blocks — they need answers before the doc is committable.

## Big-data / safety guards

- **Don't run distribution stats** here — leave that to `eda-profiler`. This skill produces structure, not data.
- **Don't include sample data** in the doc — even one example row can be PII (per `docs/data-classification.md`).
- **`DESCRIBE EXTENDED` only** — don't issue any `SELECT *` to discover columns.

## Worked example

> User: "Document `analytics_prod.core.fct_refunds` — there's no doc for it yet."

1. Read `_template.md`. Read `fct_orders.md` for the expected detail level.
2. Run `DESCRIBE EXTENDED analytics_prod.core.fct_refunds`. Output reveals columns, partition column `refund_date`.
3. Row-count check on partition `refund_date = current_date()`. Confirms ~5K rows, 5K distinct `refund_id`. Grain: one row per refund.
4. Grep usages: `rg fct_refunds --type sql --type py`. Found in `metric-definitions.md` (net revenue), `fct_orders.md` (relationship), and 12 analyses. Common filter: `WHERE refund_date >= ...`.
5. Look up classification: `customer_id` → PII; `refund_amount` → Internal.
6. Search `docs/pipelines.md` → not listed. Flag for the human.
7. Draft `docs/schema/fct_refunds.md` with:
   - Filled in: column table, partition info, relationships, sample queries, last-reviewed date.
   - `**[HUMAN: confirm]**` blocks for: business meaning of partial vs full refund, why refund_amount can be NULL, owning team, SLA, gotchas.
8. Output the `[HUMAN: confirm]` items so the user can answer them inline.

## What to add to docs/ if missing

- The producing job for this table → add to `docs/pipelines.md`.
- A column not in the canonical classification table → add to `docs/data-classification.md` under the appropriate class.
- A gotcha discovered during documentation → add to the gotchas section.
