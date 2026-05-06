---
name: plotting-graphs
description: Produce publication-quality charts that respect the data and the reader — chart-type selection, visual hierarchy, color, annotation, and the team's palette.
when_to_invoke:
  - "Plot this"
  - "Make a chart of X"
  - "Visualize the result"
  - "Build a retention curve / funnel chart / cohort heatmap"
---

# Plotting Graphs

A chart's job is to **transfer one specific insight** to a reader's brain in under a second. Almost everything about defaults — chart junk, garish colors, unsorted bars — gets in the way. This skill picks the right chart, applies the team's palette, and removes the chrome.

## Inputs

- **Data** to plot (long-format DataFrame is preferred).
- **Question type** the chart answers: comparison / composition / distribution / relationship / trend / flow.
- **Audience** (affects density and annotation level).
- Optional: title, subtitle, palette name (default: `Lavender Earth`).

## Outputs

- A figure saved to `reporting/figures/<topic>/<slug>-vN.{png,svg}` (PNG @ 2× DPI raster, SVG vector).
- Alt-text describing the **takeaway**, not the chart type.

## Procedure

1. **Pick the chart by question type:**
   - **Comparison** → bar (sorted descending unless axis is ordinal)
   - **Composition** → stacked bar / 100% stacked bar
   - **Distribution** → histogram, violin, or box (pick by sample size: hist for n>1k, box/violin for n<1k)
   - **Relationship** → scatter, hexbin (for n>10k), or heatmap (categorical × categorical)
   - **Trend** → line; aspect ratio banked toward ~45° on the dominant slope
   - **Flow / sequence** → funnel (sankey only for branching flow)
2. **Strip the chrome:**
   - No chart border, no top/right spines.
   - Mute gridlines (alpha ≤ 0.3) — keep only the ones the reader actually uses.
   - Group with whitespace, not boxes.
   - Title left-aligned. Subtitle states the **so what** ("D7 retention dropped 3pts last week"), not the chart type ("Line chart of retention over time").
3. **Typography:**
   - Title: 16pt bold, strong contrast.
   - Subtitle: 12pt medium weight, same color as body text.
   - Tick labels: ≥ 11pt.
   - Use **monospace numerals** in any in-chart tables.
   - Sentence case throughout. Title case is a 1995 affectation.
4. **Color (semantic, not decorative):**
   - Full saturation only on the **focus series**; everything else is muted.
   - Palette by data type — see palettes below.
   - Color-blind safe: test with a deuteranopia simulator (or pick from the palettes below, which have been checked).
   - Dark-mode legible: use the `Mist` palette for dark backgrounds.
5. **Annotation over legend:**
   - Direct labels next to the line/bar where space permits. Legends are visual indirection — the reader's eye has to bounce.
   - Call out **the one number** the reader should leave with (large text, contrasting color).
   - Mark events / change-points inline with a vertical rule + label.
6. **Readability:**
   - Bars sorted descending unless the axis is intrinsically ordinal (date, age band).
   - Zero baseline on bar charts — non-negotiable. Truncated bar charts are deceptive.
   - Log scale only when the range spans orders of magnitude *and* the axis is labelled "log scale."
   - For trend lines, bank to the dominant slope (~45°). Don't squash a year of data into a square.
7. **Save the output:**
   - Path: `reporting/figures/<topic>/<slug>-v<N>.{png,svg}`. Increment `N` on revision; don't overwrite.
   - PNG at 2× DPI for raster (slides, screenshots).
   - SVG for vector (decks, web docs).
   - Alt-text describes **the takeaway** ("D7 retention dropped from 28% to 25% on 2025-01-08, concentrated on iOS/Android"), not the chart type.

## Big-data / safety guards

- **Sample for visualization** when n > 100k for scatter/hex (`df.sample(frac=..., random_state=SEED)` with the seed from `docs/environment.md`).
- **Don't plot raw customer_ids or PII** — aggregate first. Run `pii-scrubber` on anything customer-level.
- **Never embed Confidential numbers** (`cost_of_goods_sold`, `gross_margin`, `lifetime_value`) in any figure destined outside the DS team.

## Color palettes

The repo defaults to a curated muted/mauve/plum/earth aesthetic. Each palette is 5 stops; use the **leftmost** as light/low and the **rightmost** as dark/high.

### Categorical — primary *(Lavender Earth)*
Default for unordered groups; widest hue range.

| Hex | Name |
|---|---|
| `#F4E8C1` | Pearl Beige |
| `#A0C1B9` | Ash Grey |
| `#70A0AF` | Pacific Blue |
| `#706993` | Vintage Lavender |
| `#331E38` | Midnight Violet |

### Categorical — warm alt *(Rose Earth)*
Softer rose-toned alternative for warmer topics.

| Hex | Name |
|---|---|
| `#AEB4A9` | Ash Grey |
| `#E0C1B3` | Almond Silk |
| `#D89A9E` | Rosy Taupe |
| `#C37D92` | Old Rose |
| `#846267` | Smoky Rose |

### Sequential — cool→warm bridge *(Icy Mocha)*
Long polychrome ramp for hierarchical / wide-range ordinal data.

| Hex | Name |
|---|---|
| `#B1DDF1` | Icy Blue |
| `#9F87AF` | Amethyst Smoke |
| `#88527F` | Grape Soda |
| `#614344` | Chocolate Plum |
| `#332C23` | Deep Mocha |

### Sequential — warm mauve *(Plum Mocha)*
Warm-leaning heatmaps / choropleths.

| Hex | Name |
|---|---|
| `#D9B8C4` | Pastel Petal |
| `#957186` | Dusty Mauve |
| `#703D57` | Mauve Shadow |
| `#402A2C` | Deep Mocha |
| `#241715` | Coffee Bean |

### Sequential — cool green *(Mint Forest)*
Cool-leaning heatmaps / choropleths.

| Hex | Name |
|---|---|
| `#B5FFE1` | Aquamarine |
| `#93E5AB` | Celadon |
| `#65B891` | Mint Leaf |
| `#4E878C` | Dark Cyan |
| `#00241B` | Evergreen |

### Sequential — lilac *(Orchid)*
Gentle single-hue ramp for low-emphasis ordinal data.

| Hex | Name |
|---|---|
| `#EFCEFA` | Thistle |
| `#D4B2D8` | Pink Orchid |
| `#A88FAC` | Lilac Ash |
| `#826C7F` | Dusty Mauve |
| `#5D4E60` | Mauve Shadow |

### Diverging
Pair `Mint Forest` (cool side) with `Plum Mocha` (warm side), interpolating through a neutral mid-stop such as `#A0C1B9` Ash Grey or `#E0C1B3` Almond Silk.

### Dark-mode *(Mist)*
Lifted greys/blues that hold contrast on dark backgrounds; use for the focus series, with `Lavender Earth`'s darker stops as supporting muted tones.

| Hex | Name |
|---|---|
| `#776D5A` | Dim Grey |
| `#987D7C` | Taupe |
| `#A09CB0` | Lilac Ash |
| `#A3B9C9` | Powder Blue |
| `#ABDAE1` | Frosted Blue |

## Worked example — retention drop chart

> Goal: show that D7 retention stepped down on 2025-01-08, concentrated in iOS/Android.

```python
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
import pandas as pd

LAVENDER_EARTH = ["#F4E8C1", "#A0C1B9", "#70A0AF", "#706993", "#331E38"]
FOCUS = "#331E38"
MUTED = "#A0C1B9"

# df: long-form with columns [d0_date, surface, d7_retention]
fig, ax = plt.subplots(figsize=(8, 4.5), dpi=200)

for surface, color in zip(
    ["web", "ios", "android"],
    [MUTED, FOCUS, "#706993"],
):
    sub = df[df.surface == surface].sort_values("d0_date")
    lw = 2.5 if surface in ("ios", "android") else 1.0
    ax.plot(sub.d0_date, sub.d7_retention, color=color, linewidth=lw, label=surface)

ax.axvline(pd.Timestamp("2025-01-08"), color="#846267", linestyle="--", linewidth=1)
ax.text(pd.Timestamp("2025-01-08"), 0.31, "  app v4.21.0", color="#846267", fontsize=10, va="top")

ax.set_title("D7 retention dropped on 2025-01-08", loc="left", fontsize=16, fontweight="bold")
ax.text(0, 1.02, "iOS and Android stepped −5 pts; Web flat", transform=ax.transAxes, fontsize=12, color="#5D4E60")

ax.set_ylabel("D7 retention")
ax.set_ylim(0.20, 0.32)
ax.yaxis.set_major_formatter(plt.matplotlib.ticker.PercentFormatter(1.0, decimals=0))
ax.xaxis.set_major_locator(mdates.WeekdayLocator())
ax.xaxis.set_major_formatter(mdates.DateFormatter("%b %d"))
for spine in ("top", "right"):
    ax.spines[spine].set_visible(False)
ax.grid(axis="y", alpha=0.25)
ax.legend(frameon=False, loc="lower left")

fig.tight_layout()
fig.savefig("reporting/figures/d7-retention/retention-drop-jan-2025-v1.png", dpi=200, bbox_inches="tight")
fig.savefig("reporting/figures/d7-retention/retention-drop-jan-2025-v1.svg", bbox_inches="tight")
```

Alt-text:
> "D7 retention dropped from ~28% to ~25% on 2025-01-08, concentrated in iOS and Android; Web is flat. The drop coincides with the app v4.21.0 release."

## What to add to docs/ if missing

- A new chart pattern the team uses repeatedly (e.g., "always plot trailing 4-week mean as a faint reference line on time series") → add as a new step in this skill's procedure.
- A new palette for a specific use case (e.g., region colors that match the brand) → add to the palettes section here.
