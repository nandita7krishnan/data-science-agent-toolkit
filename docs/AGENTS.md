# docs/AGENTS.md

This folder is the team's institutional memory. Files here are **canonical** — when there's a disagreement between a doc and what you remember from prior conversation, the doc wins.

## What lives here

| File | Purpose |
|---|---|
| `glossary.md` | Product codes, acronyms, jargon. The first thing to consult when a term is unclear. |
| `metric-definitions.md` | Exact Spark SQL formulas for every metric the team reports. **Use these, do not improvise.** |
| `business-info.md` | Business model, customer segments, key periods (e.g., holiday seasons), org structure relevant to data work. |
| `pipelines.md` | Pipeline catalog: dependencies, schedules, SLAs, failure runbooks. |
| `style-guide.md` | SQL/Python style: casing, aliasing, formatting, file layout. |
| `data-owners.md` | Domain owners, on-call contacts, escalation paths. |
| `environment.md` | Reproducibility contract: Python version, seeds, fixtures, env vars. |
| `data-classification.md` | PII / Confidential / Internal / Public — what can go where. |
| `dashboards.md` | Catalog of existing dashboards. Check here *before* building a new analysis. |
| `experiment-registry.md` | Past A/B tests: hypothesis, dates, decision, learning. Check before re-running similar experiments. |
| `schema/` | Table-by-table documentation. See `schema/AGENTS.md`. |
| `external-tables/` | Tables owned by other teams. See `external-tables/AGENTS.md`. |

## Rules for editing docs here

1. **One source of truth.** If the same fact lives in two docs, pick the most specific home and link to it from the other.
2. **Show the SQL.** Anywhere a metric, gotcha, or filter is described, paste the exact SQL snippet — not prose alone.
3. **Date your changes.** When a definition or owner changes, leave a one-line `> Updated YYYY-MM-DD: <what changed>` note at the bottom of the section.
4. **Don't delete history.** When deprecating a metric or table, mark it `**[DEPRECATED YYYY-MM-DD]**` and link to the replacement; leave the old definition for at least one quarter.
5. **Gotchas at the bottom.** Every doc section that defines something should end with a "Gotchas" subsection if any exist. This is the highest-signal part of the doc — don't bury it.

## When the agent should propose an edit here

- The user corrected you on a definition, owner, partition, or filter.
- You discovered a new gotcha while running a query.
- A pipeline schedule, SLA, or failure path changed.
- A metric was redefined or deprecated.

In all cases: propose the **exact diff**. Don't just say "I'll update the docs" — show the before/after.
