# docs/schema/AGENTS.md

Schema docs are **the agent's table reference.** When asked to write a query against a table, **read this folder first.** When the requested table isn't documented, run the `schema-documenter` skill and propose the doc — don't query blind.

## What lives here

One markdown file per first-class analytical table (`dim_*`, `fct_*`). External-team tables live in `../external-tables/`.

| File | Pattern demonstrated |
|---|---|
| `_template.md` | Copy this to start a new schema doc. |
| `fct_orders.md` | Transactional fact, partitioned by event date. The "always filter on partition" pattern. |
| `fct_page_events.md` | High-volume event stream, partitioned by event date. The "this query will bankrupt you" pattern + Monday-duplicate gotcha. |
| `dim_customers.md` | SCD Type 2 dimension. The `is_current = TRUE` filter pattern + `valid_from` / `valid_to` semantics. |

## Rules for writing/editing schema docs

1. **State the grain in the second sentence.** "One row per X per Y." If you can't, the table isn't ready to be documented — figure out the grain first.
2. **Show the partition column and the required filter** prominently. Anything below the fold is invisible.
3. **Columns table is the source of truth.** Type, nullability, allowed values, and a one-line description per column. If the column is PII or Confidential, mark it (see `../data-classification.md`).
4. **Gotchas at the bottom** — non-obvious things that bit you or could bite the next person. This is the highest-signal part of the doc.
5. **Worked example queries** — at least two: a baseline "give me a count" and a representative analytical query.
6. **Owner + last-reviewed date** at the top. If "last reviewed" is more than a quarter old and the table is actively used, the doc is suspect.

## When the agent should read which files

- **Asked to query a `fct_*` or `dim_*` table** → read `<table>.md`.
- **Asked about a relationship between tables** → read both schema docs *and* `../glossary.md` for terms.
- **Spotted a contradiction between code and docs** → flag it; don't silently follow either. Run the `query-reviewer` skill if it's a query-correctness issue.
- **Asked to document a new table** → run the `schema-documenter` skill.

## When the agent should propose an edit

- A column was added, removed, or its semantics changed.
- A new gotcha was discovered (e.g., "Mondays are duplicated due to retry logic").
- The grain changed (rare, but happens — call it out loudly).
- The partition column changed.
- An example query is wrong or out of date.

## Naming conventions for new files

- `fct_<plural_noun>.md` for fact tables (`fct_orders`, `fct_page_events`, `fct_refunds`).
- `dim_<plural_noun>.md` for dimensions (`dim_customers`, `dim_products`).
- `int_*` (intermediate) tables generally don't get their own file — document them in the consuming model's doc.
- One file per table, exactly. No combined docs.
