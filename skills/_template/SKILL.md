---
name: <kebab-case-name>
description: <one sentence: what the skill does, in present-tense imperative ("Profile any table...", "Calculate a metric...")>
when_to_invoke:
  - <trigger phrase the user is likely to say>
  - <another trigger phrase>
  - <a third>
---

# <Skill Name>

<One short paragraph: what this skill does, why it exists, and the one-line outcome the user should expect.>

## Inputs

- <Required input 1 — be explicit (e.g., "table name", "metric name from docs/metric-definitions.md")>
- <Required input 2>
- <Optional input + default>

## Outputs

- <Concrete artifact the user gets back. E.g., "A markdown table with one row per column", "An executable Spark SQL query", "A pass/fail checklist".>

## Procedure

1. <First step — usually "read the relevant doc(s)" or "ask clarifying question(s)">.
2. <Second step>.
3. ...
N. <Final step — usually "summarize and propose next action">.

## Big-data / safety guards

- <Guard against runaway cost (e.g., "Do not run on >1M rows without a partition filter")>
- <Guard against silent wrong answers (e.g., "Always print row count before and after a join")>
- <Guard against PII / data-classification violations (e.g., "Run pii-scrubber before pasting any output containing customer_id")>

## Worked example

> User: <example fuzzy ask>

Step-by-step, using your team's tables (replace with your actual table paths from `docs/schema/`):

1. <what the agent does at step 1>
2. <step 2>
3. <step 3 — ideally with a code snippet>

```sql
-- example query the skill produced
SELECT ...
FROM analytics_prod.core.<table>
WHERE ...;
```

Final output:

```
<what the user actually receives — a paragraph, table, or chart description>
```

## What to add to docs/ if missing

- If `<gap discovered>` → propose adding to `<docs/file.md>` section `<...>`.
- If `<another gap>` → propose adding to `<...>`.
