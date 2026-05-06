# `<analysis-slug>` — `<one-line title>`

> Copy this file to `analyses/YYYY-MM/<slug>/README.md` and fill in.

## The question

> The exact ask, verbatim from the stakeholder. Don't paraphrase; if you have to clarify, write the answers under "Clarifying questions" below.

## Headline

> One sentence. The answer. Filled in *after* the work is done. Goes at the top because that's what readers want.

## Clarifying questions (and their answers)

1. **<question>** → <answer>
2. ...

## Hypotheses (going in)

- **Prior:** what the team expected.
- **What would surprise me:** the result that would change behavior.

## Method

- **Tables:** `<list>` (link to `docs/schema/<table>.md` for each).
- **Metrics:** `<list>` (link to `docs/metric-definitions.md` entries).
- **Time window:** `<from>` to `<to>` (timezone: UTC).
- **Exclusions:** `<list — typically: synthetic, internal traffic, test accounts, employees, anonymous customers depending on metric>`.
- **Approach:** `<descriptive | cohort | funnel | root-cause | experiment | predictive>`.

## Results

- Bullet 1 — fact + interpretation.
- Bullet 2 — fact + interpretation.
- ...

## Caveats

- `<caveat 1 — be explicit>`
- `<caveat 2>`

## Recommended next step

- `<concrete action — decision, follow-up question, additional analysis, ship>`

## Files

- `notebook.ipynb` — the work.
- `figures/` — saved charts (if any).

## Reviewer

- Reviewed by: `<name or notebook-reviewer skill>`
- Review comments addressed: `<yes / which ones still open>`
