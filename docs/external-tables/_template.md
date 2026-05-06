# `<catalog.schema.stg_<source>__<entity>>`

> Replace every `<...>` placeholder. Delete this line before committing.

## Ownership & access

- **Owning team:** `<Team Name>` (Slack: `<#channel>`)
- **Source system:** `<CRM | ERP | ad platform | external SaaS — name of the system>`
- **Access level:** `<read for everyone in <default-readers-group> | request via <flow>>`
- **SLA:** `<freshness commitment, e.g., "available by 07:00 UTC daily; max delay 2h">`
- **Bug-report path:** `<Jira project / intake form / channel — NOT a DM>`
- **Last reviewed:** `<YYYY-MM-DD>` by `<name>`

## Overview

- **Full path:** `<catalog.schema.table>`
- **Grain:** `<one row per X per Y>`
- **Type:** `<staging | fact | dimension>` (almost always `staging` for external tables)

## Partitioning & filters

- **Partition column(s):** `<column>`
- **Required filter:** `WHERE <partition_column> >= <example>`
- **Recommended date range:** `<...>`

## Freshness

- **Update frequency:** `<...>`
- **Typical lag:** `<...>`
- **How to check:** `SELECT MAX(<col>) FROM <full_path>;`
- **Producing job (theirs, not ours):** `<job name + link if available>`

## Columns

| Column | Type | Nullable | Class | Description |
|---|---|---|---|---|
| `<col>` | `<TYPE>` | `<Y/N>` | `<class>` | `<one line>` |

### Allowed values for categorical columns

- `<col>`: `<value1 | value2 | ...>`

## Relationships

- **Joins to (our internal tables):** `<our_table.our_col> = <this_table.col>` (kind: `<1:1 / 1:N / N:M>`)

## Gotchas

- `<gotcha 1>`
- `<gotcha 2>`

## Example queries

### Sanity check

```sql
SELECT MAX(<col>) AS latest, COUNT(*) AS rows_today
FROM <full_path>
WHERE <partition_column> = current_date();
```

### Representative analytical query (joining with our internal tables)

```sql
-- Describe what this answers in one line.
SELECT ...
FROM <full_path> ext
JOIN analytics_prod.core.<our_table> internal
  ON ...
WHERE ...;
```

## Migration / deprecation notes

- `<note>`

## Change log

- `YYYY-MM-DD` (`<author>`): `<what changed>`
