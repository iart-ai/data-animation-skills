# Chart Animation

Turn a dataset into a chart that moves with intent — a bar chart race, a growing line, a count-up stat — with every value driven from the current frame so numbers stay exact and the render never desyncs. For analysts, PMs, and data storytellers who want accurate, on-brand data motion instead of a fragile screen recording.

## When it activates

- "Make a bar chart race" or "animate a ranking over time."
- "Turn a CSV into a video" or "build an animated data visualization."
- "Make a number counter / ticker" or "animate a chart/graph."
- "Batch-render one chart template across many datasets."

## Example prompts

- "Turn this CSV of yearly revenue by region into a bar chart race with a moving year label."
- "Animate a counter from 0 to $1.2M, rounded and formatted as currency."
- "Render this line-chart template for every dataset in my /data folder."

## What's inside

- `SKILL.md` — frame-driven value interpolation, value scales vs. frame interpolators, easing rules, counters/tickers, rank transitions, pacing budgets, labeling, and the template × data batch pattern.
- `references/bar-chart-race.md` — a complete runnable Remotion bar-chart-race component (sparse-row keyframe interpolation, per-frame ranking, gliding y-positions, animated value labels, D3-scale axis) plus a vanilla canvas variant.
- `references/data-pipeline.md` — CSV/JSON → typed props parsing and validation, the theme object pattern, animated counter/line-reveal recipes, and the template × data batch render script.

---
Part of **[Data Video Skills](../)** · Built by **[iart.ai](https://iart.ai)** — controllable Motion Graphics MP4 from a prompt or data.
