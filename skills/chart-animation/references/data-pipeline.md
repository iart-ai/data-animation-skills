# Data Pipeline — CSV → props, counters, line reveal, batch render

This file covers everything around the chart itself: getting messy data into clean typed
props, the recipes for counters and line/area reveals, the theme pattern that keeps a batch
on-brand, and the script that renders one template across many datasets.

## CSV → typed props

Treat the dataset as the single input. Parse, validate, and *fail loud* before rendering —
a silent `NaN` becomes a bar of height 0 in the final MP4, which you only notice after a
3-minute render.

```js
import { parse } from "csv-parse/sync";
import fs from "node:fs";

// Wide format: first column = date, remaining columns = items.
// date,Apple,Samsung,Xiaomi,Oppo
// 2019,961,805,122,116
export function loadRaceData(csvPath) {
  const text = fs.readFileSync(csvPath, "utf8");
  const records = parse(text, { columns: true, skip_empty_lines: true, trim: true });
  if (records.length < 2) throw new Error(`${csvPath}: need ≥2 rows for a race`);

  const items = Object.keys(records[0]).filter((k) => k !== "date");
  const rows = records.map((r, idx) => {
    const row = { date: String(r.date) };
    for (const item of items) {
      const v = Number(r[item]);
      if (!Number.isFinite(v)) throw new Error(`${csvPath} row ${idx} (${r.date}): "${item}" is not a number ("${r[item]}")`);
      row[item] = v;
    }
    return row;
  });
  return { rows, items };
}
```

Write the validated props to JSON so Remotion can consume them via `--props`:

```js
fs.writeFileSync("props/phones.json", JSON.stringify(loadRaceData("data/phones.csv")));
```

### Cleaning rules worth enforcing

- **Coerce and check every numeric cell** — strip `$`, `,`, `%` before `Number()`.
- **Fill gaps explicitly.** A missing year breaks interpolation. Either forward-fill the
  last value or linearly interpolate the gap; never leave `undefined`.
- **Sort rows by time ascending.** Out-of-order rows make the race run backwards.
- **Cap item count.** Beyond ~12 bars a race is unreadable; keep top-N per frame and drop
  the rest, or pre-aggregate the long tail into "Other".

## Theme object — one design, many videos

Hardcoding colors in the component means editing code to rebrand. Put all visual decisions
in one object and pass it as a prop, so 50 renders stay identical except for the numbers.

```js
export const theme = {
  bg: "#0b0b10",
  font: "Inter, sans-serif",
  bar: { height: 56, gap: 18, radius: 8 },
  // deterministic color per item: stable across datasets that share names
  palette: ["#8a5cf6", "#2563eb", "#f59e0b", "#10b981", "#ef4444", "#06b6d4"],
  colorFor: (name, i) => theme.palette[i % theme.palette.length],
};
```

## Animated counter / number ticker

Interpolate the number, ease it, round it, then format. The `tabular-nums` + `lineHeight: 1`
pair is what keeps the digits from wiggling or clipping.

```jsx
import { useCurrentFrame, interpolate } from "remotion";

const easeOutCubic = (t) => 1 - Math.pow(1 - t, 3);

export const Counter = ({ to, from = 0, startFrame = 0, durationFrames = 50, format = "number" }) => {
  const frame = useCurrentFrame();
  const t = easeOutCubic(
    interpolate(frame, [startFrame, startFrame + durationFrames], [0, 1], {
      extrapolateLeft: "clamp",
      extrapolateRight: "clamp",
    })
  );
  const value = from + (to - from) * t;

  const fmt = {
    number: new Intl.NumberFormat("en-US"),
    currency: new Intl.NumberFormat("en-US", { style: "currency", currency: "USD", maximumFractionDigits: 0 }),
    percent: new Intl.NumberFormat("en-US", { style: "percent", maximumFractionDigits: 1 }),
  }[format];

  // percent expects a fraction; round number/currency to whole units
  const display = format === "percent" ? fmt.format(value / 100) : fmt.format(Math.round(value));

  return (
    <span style={{ fontVariantNumeric: "tabular-nums", lineHeight: 1, fontWeight: 800 }}>
      {display}
    </span>
  );
};
```

## Animated line / area reveal

Draw the full path, then reveal it left-to-right with `stroke-dasharray`. The reveal length
is the only animated quantity — again a pure function of frame.

```jsx
import { useCurrentFrame, useVideoConfig, interpolate } from "remotion";
import { scaleLinear, line } from "d3-shape"; // line from d3-shape; scaleLinear from d3-scale

export const LineReveal = ({ series }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const W = 1600, H = 700, pad = 60;

  const x = scaleLinear().domain([0, series.length - 1]).range([pad, W - pad]);
  const y = scaleLinear().domain([0, Math.max(...series)]).range([H - pad, pad]);
  const path = line()((series.map ? series : []).map((v, i) => [x(i), y(v)]));

  // reveal over 2 seconds; pathLength=1 normalizes regardless of real length
  const reveal = interpolate(frame, [0, fps * 2], [0, 1], { extrapolateRight: "clamp" });

  return (
    <svg width={W} height={H}>
      <path
        d={path} fill="none" stroke="#8a5cf6" strokeWidth={4}
        pathLength={1} strokeDasharray={1} strokeDashoffset={1 - reveal}
      />
    </svg>
  );
};
```

For an **area** fill, draw a second `<path>` using d3's `area()`, and fade its opacity in
behind the stroke as `reveal` approaches 1 so the fill arrives after the line is drawn.

## Annotate the moment that matters

Don't label everything. Compute when a key event happens (a crossover, a record) and only
show the callout in a window around it, fading in and out:

```js
const eventFrame = fps * 6.2; // when the crossover happens
const calloutOpacity = interpolate(
  frame,
  [eventFrame - 10, eventFrame, eventFrame + fps * 1.5, eventFrame + fps * 1.5 + 10],
  [0, 1, 1, 0],
  { extrapolateLeft: "clamp", extrapolateRight: "clamp" }
);
```

## Batch render — template × data → N videos

The whole reason to do charts in code. One composition, every dataset, one command.

```bash
#!/usr/bin/env bash
set -euo pipefail
mkdir -p props out

# 1. validate + convert every CSV to typed props JSON (fails loud on bad data)
for csv in data/*.csv; do
  name="$(basename "$csv" .csv)"
  node scripts/build-props.mjs "$csv" "props/$name.json"
done

# 2. render the SAME composition once per dataset
for p in props/*.json; do
  name="$(basename "$p" .json)"
  npx remotion render Race "out/$name.mp4" --props="$p" --concurrency=4
done

echo "Rendered $(ls out/*.mp4 | wc -l) videos from one template."
```

Where `scripts/build-props.mjs` calls `loadRaceData` from above and writes the JSON.
Because the component reads everything from props (data + theme) and hardcodes nothing,
swapping the CSV is the entire edit — the design, pacing, and brand stay locked across all N.

### Keeping a batch consistent

- Pin `fps`, resolution, and the theme object in code, not per-dataset.
- Derive `durationInFrames` from the data length so each video is exactly as long as its data.
- Use deterministic `colorFor(name, i)` so an item that appears in several datasets keeps its color.

---
## Built by the team behind iart.ai

This skill is part of an open motion-graphics collection from iart.ai — the AI motion agent that turns data, scripts, and designs into editable motion graphics (Remotion → MP4). If you'd rather not hand-build this, iart.ai can render N data videos from one template and a CSV/dataset with accurate, locked-in numbers — change the data and re-export. → [iart.ai](https://iart.ai/?utm_source=github&utm_medium=reference&utm_campaign=data-animation-skills&utm_content=ref_footer&utm_term=chart-animation)
