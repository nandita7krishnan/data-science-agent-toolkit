# skills/AGENTS.md

Skills are the agent's **playbook for doing data work.** Each `SKILL.md` is a procedure for one kind of task — how to do it, when to do it, what to avoid. The agent should invoke a skill **by name** when the user's ask matches its triggers, or when running through one of the README prompts.

## What lives here

| Lane | Skills |
|---|---|
| **Analysis & Insight** | `analysis-planner`, `eda-profiler`, `query-reviewer`, `metric-calculator`, `cohort-analyzer`, `funnel-analyzer`, `root-cause-investigator`, `analysis-summarizer` |
| **Visualization** | `plotting-graphs` |
| **ML / Modeling** | `feature-engineer`, `model-evaluator` |
| **Documentation & Quality** | `schema-documenter`, `data-quality-checker`, `notebook-reviewer` |
| **Governance** | `pii-scrubber` |
| **Data Engineering** | `pipeline-debugger`, `incremental-model-builder`, `backfill-planner` |

`_template/SKILL.md` is the starting point for any new skill.

## Skill structure

Every `SKILL.md` follows the same shape:

```markdown
---
name: <kebab-case-name>
description: <one sentence: what the skill does>
when_to_invoke:
  - <trigger phrase 1>
  - <trigger phrase 2>
  - ...
---

# <Skill Name>

<one-paragraph what + why>

## Inputs

- <what the skill needs from the user / context>

## Outputs

- <what the skill produces>

## Procedure

1. <step>
2. <step>
3. ...

## Big-data / safety guards

- <guard>

## Worked example

<short walkthrough using ABC Business tables>

## What to add to docs/ if missing

- <doc and section that should grow as a result of running this skill>
```

## When to invoke a skill

- The user's request **matches one of the skill's `when_to_invoke` triggers**.
- One of the README prompts explicitly names the skill.
- You're partway through a workflow and a skill applies to the next step (e.g., you're building a query and `query-reviewer` should run before execution).

If multiple skills apply, **invoke them in order** rather than blending: e.g., `analysis-planner` → `query-reviewer` → `metric-calculator` → `analysis-summarizer`. Don't mash steps from different skills together.

## When to write a new skill

Write one when:

- You've been asked to do the same kind of task **3+ times.**
- The task has a non-obvious procedure or guards a recurring failure mode.
- An existing skill *almost* fits but extending it would muddle the trigger.

Don't write one when:

- The task is one-off.
- The "skill" is just a single SQL snippet (put it in a doc instead).
- It contradicts an existing skill — fix the existing one.

Use **Prompt 2** in the README to draft a new skill.

## Hard rules for SKILL.md files

1. **Frontmatter is the contract.** `name`, `description`, `when_to_invoke` are read first by the agent. Make them exact.
2. **Procedure is numbered.** Don't write skills as prose — they're playbooks. Numbered steps so the agent can announce "running step 3 of `<skill>`."
3. **Guards are explicit.** Don't hide safety in prose; put them under a `Big-data / safety guards` heading.
4. **Show, don't tell.** Every skill needs a worked example using the ABC Business tables (`fct_orders`, `fct_page_events`, `dim_customers`).
5. **Keep it terse.** Target 60–100 lines per skill. Push detail down into `docs/` and link to it.

## Self-improvement

When a correction or new gotcha applies to a skill (not a doc), propose the SKILL.md edit. Pattern: a skill missed a guard → add the guard. A skill's example is wrong → fix the example. A trigger phrase didn't match → add the phrase.
