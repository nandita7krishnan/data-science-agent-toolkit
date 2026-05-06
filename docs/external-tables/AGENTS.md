# docs/external-tables/AGENTS.md

External tables are tables **owned by other teams** that we read but do not produce. The rules differ from internal tables in three important ways:

1. **You can't change the schema.** Schema changes go through the owning team's intake.
2. **The SLA is theirs, not ours.** When this table is late, you wait — you don't fix it.
3. **The owner contact must be loud.** Every doc in this folder needs the owner's team and Slack channel at the top.

## What lives here

One markdown file per externally-owned table we frequently consume. Use `_template.md` as the starting point.

| File | Pattern demonstrated |
|---|---|
| `_template.md` | Copy this for any new external table doc. |
| `stg_marketing__campaigns.md` | Marketing-Analytics-owned campaign metadata. The "cross-team naming convention + SLA + owner" pattern. |

## Naming convention

External tables follow `stg_<source_team>__<entity>` (double underscore separating source from entity):

- `stg_marketing__campaigns`
- `stg_marketing__attribution`
- `stg_finance__chart_of_accounts`
- `stg_inventory__stock_levels`

The double-underscore is intentional — it makes the source team grep-able.

## Required sections (in addition to schema doc requirements)

Every file in this folder must include, **at the top** (above the column table):

- **Owning team** — full team name + Slack channel.
- **Source system** — what produces it (CRM, ERP, ad platform export, etc.).
- **Access level** — who can read it; if it requires a special grant, document the request flow.
- **SLA** — owning team's commitment for freshness and availability.
- **Bug-report path** — where to report data quality issues. Usually a Jira intake project, **not** a DM to a specific person.

## Rules for editing

- **Don't add columns to the column table that don't exist in the source.** Run `DESCRIBE EXTENDED <full_path>` and use that as truth.
- **Do** add gotchas, especially business-context gotchas the source team's docs miss (they often do).
- **When the owning team renames or retires a column**, mark it `**[DEPRECATED]**` here, link to the replacement, and notify any consumer code in this repo (search by the old column name).

## When the agent should propose an edit

- Discovered a new gotcha while consuming the table.
- The owning team announced a schema change in their channel — capture the migration plan here.
- The SLA changed.

## When the agent should *not* propose an edit

- The source data looks wrong → that's a bug report to the owning team, not a doc edit.
- You disagree with the owning team's column naming → that's a feedback discussion, not a unilateral doc change.
