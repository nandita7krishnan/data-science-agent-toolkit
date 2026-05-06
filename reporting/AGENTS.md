# reporting/AGENTS.md

This folder is for **recurring reports and dashboards** consumed by stakeholders. The catalog is in `docs/dashboards.md`; the implementation lives here.

Things that generate insight-on-demand for a person live here. Things that produce data for a system live in `pipelines/`. Ad-hoc analyses live in `analyses/`.

## Folder structure

```
reporting/
├── AGENTS.md
├── <report-slug>/
│   ├── README.md            (audience, refresh, owner, decisions it supports)
│   ├── query.sql            (or notebook.ipynb if interactive)
│   ├── figures/             (saved snapshots, optional)
│   └── runbook.md           (what to do when the dashboard looks wrong)
└── figures/<topic>/         (versioned saved figures from analyses worth keeping)
```

## Conventions

1. **Audience first.** Every recurring report has a clear primary audience (exec, PM team, marketing, etc.). The audience drives length, density, format. Run `analysis-summarizer` for the right shape.
2. **Stable URL / cell.** If the report is a dashboard in a BI tool, the dashboard URL goes in `docs/dashboards.md` and is referenced from `<report-slug>/README.md`.
3. **Numbers come from `docs/metric-definitions.md` formulas.** No improvising. If a report needs a metric not yet defined, **define it first** (Prompt 4 in README.md), don't ship the report.
4. **Data-classification respected.** No raw PII, no Confidential numbers in reports with audiences outside the DS team. Run `pii-scrubber` on output.
5. **Figures use the team palette.** Per `skills/plotting-graphs/SKILL.md`. Default `Lavender Earth` for categorical, `Plum Mocha` / `Mint Forest` for sequential.

## Adding a new report

1. **Question the recurrence.** Will this actually be looked at every week / month? If it's "just to have on a dashboard," the maintenance cost outweighs the value. Most "we should have a dashboard for X" don't survive the second month.
2. **Check `docs/dashboards.md`.** Maybe it already exists.
3. **Catalog entry first.** Add to `docs/dashboards.md` before building.
4. **Build it.** Use the appropriate skill (`metric-calculator`, `cohort-analyzer`, `funnel-analyzer`, etc.) for the underlying analysis. Use `plotting-graphs` for figures.
5. **Write the runbook.** What does it mean if the report shows zeros / NaNs / numbers way off? Who do you ping?

## Modifying an existing report

- Numbers should not change without communication. If you fix a methodology bug, the change is **a release** — note it in the report's footer with a date and a link to the change.
- Audience expectations drift. A monthly report becoming "the report" for a weekly meeting requires a refresh-frequency bump and an owner re-confirmation.

## Killing a report

When a report stops being looked at (check the BI tool's view counts, ask the audience):

1. Mark it **`[DEPRECATED YYYY-MM-DD]`** in `docs/dashboards.md`. Don't delete the entry.
2. Move the implementation to `reporting/_archive/<slug>/` (or the equivalent in your BI tool).
3. Notify any remaining viewers via the dashboard channel.
4. After 90 days with no complaints, fully retire.

## Saved figures

Static figures (e.g., charts you put in a quarterly deck) live in `reporting/figures/<topic>/<slug>-vN.{png,svg}` per `skills/plotting-graphs/SKILL.md`. Increment the version on revision; never overwrite. Source data + the script that produced the figure should be reproducible from the analysis it came from.
