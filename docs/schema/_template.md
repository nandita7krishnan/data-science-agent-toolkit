# `<catalog.schema.table_name>`

> Replace every `<...>` placeholder. Delete this line before committing.

## Overview

- **Full path:** `<catalog.schema.table_name>`
- **Owner:** `<team>` (Slack: `<#channel>`)
- **Grain:** `<one row per X per Y>`. **State this clearly.** If you can't, stop — the table isn't ready to document.
- **Type:** `fact | dimension (SCD1) | dimension (SCD2) | snapshot | bridge`
- **Last reviewed:** `<YYYY-MM-DD>` by `<name>`

## Partitioning & filters

- **Partition column(s):** `<column>` (`<DATE | TIMESTAMP | STRING>`)
- **Required filter:** `WHERE <partition_column> >= <example>`. Full scan is **`<size>`** — don't do it.
- **Recommended date range:** `<e.g., last 90 days for most analyses>`
- **Clustering / Z-order:** `<columns, if any>`

## Freshness

- **Update frequency:** `<e.g., hourly, daily at 02:00 UTC>`
- **Typical lag (event-time → table):** `<e.g., 90 min>`
- **Watermark:** `<e.g., 6 hours — late-arriving rows beyond this go to <table>_late>`
- **How to check:** `SELECT MAX(<partition_column>) FROM <full_path>;`
- **Producing job:** `<job name>` (see `docs/pipelines.md`)

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `<col>` | `<TYPE>` | `<Y/N>` | `<Public/Internal/Confidential/PII>` | `<one line>` |

> **Class** = the data classification per `docs/data-classification.md`. **PII columns must be scrubbed** before sharing output externally.

### Allowed values for categorical columns

- `<col>`: `<value1 | value2 | value3>` (deprecated: `<old_value>`)

## Relationships

- **Joins to:** `<table.column> = <other_table.column>` (kind: `<1:1 / 1:N / N:M>`)
- **Joining on `<dim_table>`** requires the `is_current = TRUE` filter (SCD2). See `dim_customers.md` for the pattern.

## Gotchas

> The most important section in the doc. List anything that has bitten or could bite the next person.

- **`<gotcha 1>`:** what it is, when it triggers, and how to handle it.
- **`<gotcha 2>`:** ...

## Example queries

### Sanity check — does the table have today's data?

```sql
SELECT MAX(<partition_column>) AS latest, COUNT(*) AS rows_today
FROM <full_path>
WHERE <partition_column> = current_date();
```

### Representative analytical query

```sql
-- Describe what this answers in one line.
SELECT
    <key_dimension>
  , COUNT(*) AS row_count
FROM <full_path>
WHERE <partition_column> >= date_sub(current_date(), 30)
GROUP BY <key_dimension>
ORDER BY row_count DESC
LIMIT 100;
```

## Change log

- `YYYY-MM-DD` (`<author>`): `<what changed>`
