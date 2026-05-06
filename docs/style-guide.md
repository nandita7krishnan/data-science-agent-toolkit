# Style Guide

How we write SQL and Python in this repo. Goal: readable in code review without a debugger, debuggable without re-indenting, and grep-able.

> Forking? Most of this is portable. The SQL section assumes Spark SQL — adapt if you're on Snowflake/BigQuery/DuckDB.

---

## SQL

### Layout

- **One clause per line** for top-level keywords (`SELECT`, `FROM`, `WHERE`, `GROUP BY`, `ORDER BY`, `HAVING`).
- **Lower-case keywords**? Either is fine — pick one and stay consistent. **This repo uses UPPER-case keywords**, lower-case identifiers.
- **Lead commas**, not trailing — easier to comment out a column without breaking the prior line:

```sql
SELECT
    customer_id
  , order_date
  , order_total
FROM analytics_prod.core.fct_orders
WHERE order_date >= DATE '2025-01-01';
```

- **CTEs over nested subqueries.** Once the query has more than one logical step, name each step with a CTE.
- **Indent join `ON` to match the join keyword.** `LEFT JOIN ... ON ...` on one line if short, otherwise:

```sql
LEFT JOIN analytics_prod.core.dim_customers c
  ON c.customer_id = o.customer_id
 AND c.is_current = TRUE
```

### Naming

- **`snake_case`** everywhere — table names, column names, CTE names.
- **`fct_*` / `dim_*` / `stg_*` / `int_*`** prefixes per `docs/glossary.md`.
- **Plural for fact tables** (`fct_orders`, not `fct_order`). Singular for dimensions (`dim_customer` ↔ `dim_customers` — repo uses **plural** for dims too, for consistency).
- **CTE names describe the result, not the operation.** `customers_with_first_order`, not `step1_join`.
- **Aliases:** single letter is fine for ≤ 2 tables. For 3+, use 2–3-letter mnemonics (`o` for orders, `c` for customers, `pe` for page_events).

### Filters

- **Always filter on the partition column first.** It's the difference between a fast query and an expensive one.
- **Use date literals**: `DATE '2025-01-15'`, not `'2025-01-15'` (avoids implicit-cast surprises).
- **Use `BETWEEN` for inclusive ranges**, `>=` and `<` for half-open. Be explicit which one you mean.
- **`IS NULL` / `IS NOT NULL` only** — never `= NULL`.

### Joins

- **Default to `LEFT JOIN`** when you want all rows from the left side. **Default to `INNER JOIN`** when you only want matched rows. Don't use `JOIN` (ambiguous) — write `INNER JOIN`.
- **Validate row counts before and after every join.** Print them — don't trust your mental model.
- **No `CROSS JOIN`** without a comment explaining why.
- **No implicit Cartesian via comma-join syntax.** If you see `FROM a, b WHERE`, rewrite as an explicit join.

### Aggregation

- **`COUNT(DISTINCT col)` is exact.** Don't use `approx_count_distinct` unless cardinality is in the millions and a 1–2% error is acceptable — and call it out in the comment.
- **`GROUP BY` ordinals are okay for short queries**, but column names are clearer in long ones. Pick one and stay consistent within a file.
- **Always `ORDER BY` before claiming a result is "the top N."**

### Parameterization (Spark SQL)

```python
spark.sql("""
  SELECT order_date, COUNT(*) AS orders
  FROM analytics_prod.core.fct_orders
  WHERE order_date >= :start_date
    AND status = :status
  GROUP BY order_date
""", args={"start_date": "2025-01-01", "status": "completed"})
```

- **Never** build SQL with `f"... WHERE order_date = '{date}'"`. Always parameterize.

### Comments

- One-line comment **above** the clause it explains, not at the end of a 100-char line.
- Use `--` for SQL comments. `/* */` only for multi-line.
- Don't comment what the code does; comment **why** it does it that way:

```sql
-- Exclude internal IPs (see docs/glossary.md "internal traffic")
WHERE ip_address NOT IN (SELECT ip_address FROM analytics_prod.core.dim_internal_ips)
```

---

## Python

### Imports

- Standard order: stdlib → third-party → local. One blank line between groups.
- **No `from x import *`.** Ever.
- **Top-of-file or top-of-cell only.** No mid-function imports unless lazy-loading a heavy dep.

### Naming

- `snake_case` for variables, functions, modules.
- `PascalCase` for classes.
- `UPPER_SNAKE` for module-level constants.
- DataFrame variables: `df_<noun>` (e.g., `df_orders`, `df_customers_with_lifetime`). Never just `df`, except inside a 5-line function with no other dataframes.

### DataFrames

- **Method chaining** for ≤ 4 steps; assign to a named variable when you cross 4.
- **Always print row counts** at material checkpoints (after a join, after a filter that drops rows).
- **Don't `.toPandas()` on > 1M-row Spark DataFrames** — sample or aggregate first.
- **Use parameterized SQL** (above) over `df.filter(F.col(...) == ...)` for anything more complex than 2 conditions. SQL is easier to review.

### Functions

- One thing per function. If you can't name it without "and," split it.
- **Type-hint everything public.** Argument types and return type. `pyspark.sql.DataFrame` is fine.
- **Docstring** on any function > 5 lines: one-sentence summary, then `Args` / `Returns` sections.

### Notebooks

- See `analyses/AGENTS.md` for the structure (header, setup, load, analyze, summarize).
- **Cells must execute top-to-bottom** in a fresh kernel. Run `Restart & Run All` before considering a notebook done.
- **No print-debug leftovers.** Remove `print(df.head())` cells before commit.
- **Set the seed in the setup cell** (see `docs/environment.md`).

### Errors

- **Don't catch and ignore.** If you catch an exception, either re-raise after logging or handle it explicitly with a comment explaining why.
- **`assert` is for invariants in dev**; use `if not ...: raise ValueError(...)` in production code paths.

---

## Linting & formatting

- **Ruff** — line length 100, `py312` target.
- **`uv run ruff check .`** must pass before commit.
- **No autoformatters on SQL** (sqlfluff is fine if you want it; not enforced here). Hand-format using the rules above.

## Naming examples — what we like / what we don't

| ✗ Avoid | ✓ Prefer |
|---|---|
| `df1`, `df_temp`, `data` | `df_orders`, `df_orders_with_customer` |
| `SELECT * FROM orders` | `SELECT order_id, customer_id, order_total FROM analytics_prod.core.fct_orders` |
| `LEFT JOIN customers ON o.id = c.id` | `LEFT JOIN analytics_prod.core.dim_customers c ON c.customer_id = o.customer_id AND c.is_current = TRUE` |
| `WHERE date > '2025-01-01'` | `WHERE order_date >= DATE '2025-01-01'` |
| `def calc(x, y): ...` | `def compute_d7_retention(d0_users: list[str], d7_users: list[str]) -> float: ...` |
