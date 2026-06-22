# Bar Chart Race — complete implementations

A bar chart race ranks a set of items over time and animates both their values and
their positions. The two hard parts:

1. **Sparse data.** Real datasets have one row per year/month. Playing one row per second
   looks like a slideshow. The fix is to interpolate continuously *between* rows.
2. **Reordering.** When item B overtakes item A, the bars must glide past each other,
   not teleport. The trick: derive each bar's y-position from an **interpolated rank**,
   so position is just another value you tween.

## Data shape

Assume time-series keyed by item, one object per timestamp:

```js
// rows[i] = the state at keyframe i; keys are item names, values are numbers
const rows = [
  { date: "2019", Apple: 961, Samsung: 805, Xiaomi: 122, Oppo: 116 },
  { date: "2020", Apple: 1043, Samsung: 933, Xiaomi: 146, Oppo: 144 },
  { date: "2021", Apple: 1188, Samsung: 901, Xiaomi: 191, Oppo: 143 },
  { date: "2022", Apple: 1287, Samsung: 870, Xiaomi: 188, Oppo: 130 },
];
const items = ["Apple", "Samsung", "Xiaomi", "Oppo"];
```

## Remotion component (the recommended path)

Deterministic, server-rendered, exports clean MP4. Every value derives from
`useCurrentFrame()` — there are zero library timers, so it never flickers.

```jsx
import { useCurrentFrame, useVideoConfig, interpolate } from "remotion";
import { scaleLinear } from "d3-scale";

const COLORS = { Apple: "#8a5cf6", Samsung: "#2563eb", Xiaomi: "#f59e0b", Oppo: "#10b981" };
const easeInOut = (t) => (t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2);
const fmt = new Intl.NumberFormat("en-US");

// Seconds spent per keyframe pair. 1.5s reads as a brisk-but-trackable race.
const SECONDS_PER_ROW = 1.5;

export const BarChartRace = ({ rows, items }) => {
  const frame = useCurrentFrame();
  const { fps, width, height } = useVideoConfig();

  const barH = 56;
  const gap = 18;
  const chartW = width - 360; // leave room for labels left + values right
  const left = 220;

  // 1. frame → continuous position across the sparse rows
  const tRow = frame / (fps * SECONDS_PER_ROW);
  const i = Math.min(Math.floor(tRow), rows.length - 2);
  const f = easeInOut(Math.min(Math.max(tRow - i, 0), 1));

  // 2. interpolate every item's value at this exact moment
  const data = items.map((name) => ({
    name,
    value: rows[i][name] + (rows[i + 1][name] - rows[i][name]) * f,
  }));

  // 3. rank by interpolated value, then derive y from rank (so overtakes glide)
  data.sort((a, b) => b.value - a.value);
  data.forEach((d, rank) => { d.y = rank * (barH + gap); });

  // 4. x-scale relative to the current leader, so the longest bar fills the frame
  const maxValue = Math.max(...data.map((d) => d.value));
  const x = scaleLinear().domain([0, maxValue]).range([0, chartW]);

  // interpolated date label ("2021" → "2022" as f crosses)
  const dateLabel = f < 0.5 ? rows[i].date : rows[i + 1].date;

  return (
    <div style={{ width, height, background: "#0b0b10", fontFamily: "Inter, sans-serif", position: "relative" }}>
      {data.map((d) => (
        // STABLE KEY per item is essential — it lets React keep each bar's DOM node
        // across reorders so the browser/renderer doesn't recycle nodes mid-glide.
        <div
          key={d.name}
          style={{
            position: "absolute",
            top: 120 + d.y,
            left,
            height: barH,
            width: x(d.value),
            background: COLORS[d.name],
            borderRadius: 8,
            display: "flex",
            alignItems: "center",
            justifyContent: "flex-end",
            color: "#fff",
            fontWeight: 700,
            fontVariantNumeric: "tabular-nums", // digits don't wiggle
            paddingRight: 16,
          }}
        >
          <span style={{ position: "absolute", right: "calc(100% + 16px)", whiteSpace: "nowrap" }}>
            {d.name}
          </span>
          {fmt.format(Math.round(d.value))}
        </div>
      ))}
      <div style={{ position: "absolute", bottom: 60, right: 80, fontSize: 96, fontWeight: 800, color: "#ffffff22", fontVariantNumeric: "tabular-nums" }}>
        {dateLabel}
      </div>
    </div>
  );
};
```

Compose it with an intro hold and a final hold so the data lands:

```jsx
import { Composition } from "remotion";
import { BarChartRace } from "./BarChartRace";
import data from "./data/phones.json"; // { rows, items }

const SECONDS_PER_ROW = 1.5, fps = 30, hold = 2; // seconds of final hold
const durationInFrames = Math.round((data.rows.length - 1) * SECONDS_PER_ROW * fps + hold * fps);

export const Root = () => (
  <Composition
    id="Race" component={BarChartRace}
    durationInFrames={durationInFrames} fps={fps} width={1920} height={1080}
    defaultProps={{ rows: data.rows, items: data.items }}
  />
);
```

### Why each choice matters

- **Rank-derived y, not data-index y.** If y came from the original array order, bars would
  jump when the sort changed. Deriving y from the freshly-sorted rank each frame makes
  position a continuous, tween-able quantity.
- **`easeInOut` on the inter-row progress** gives each transition a gentle start/stop, so
  the race breathes between years instead of moving at constant speed.
- **`extrapolateRight: "clamp"`** (used elsewhere) and clamping `f` prevent values shooting
  past the final row on the last frame.
- **Leader-relative x-scale** keeps the frame full: the top bar always spans the chart,
  and others scale against it, which exaggerates the gaps that make a race exciting.

## Vanilla canvas variant (no React)

When embedding in a plain page or rendering frames with `node-canvas`, the same math applies —
just draw instead of returning JSX. Drive it from an explicit frame counter, not `Date.now()`,
so it stays reproducible.

```js
function drawFrame(ctx, frame, { rows, items, fps = 30, secondsPerRow = 1.5, W, H }) {
  const easeInOut = (t) => (t < 0.5 ? 2*t*t : 1 - Math.pow(-2*t+2, 2)/2);
  const tRow = frame / (fps * secondsPerRow);
  const i = Math.min(Math.floor(tRow), rows.length - 2);
  const f = easeInOut(Math.min(Math.max(tRow - i, 0), 1));

  const data = items
    .map((name) => ({ name, value: rows[i][name] + (rows[i+1][name] - rows[i][name]) * f }))
    .sort((a, b) => b.value - a.value);

  const max = Math.max(...data.map((d) => d.value));
  const barH = 48, gap = 16, left = 220, chartW = W - 360;
  ctx.clearRect(0, 0, W, H);
  ctx.fillStyle = "#0b0b10"; ctx.fillRect(0, 0, W, H);

  data.forEach((d, rank) => {
    const y = 120 + rank * (barH + gap);
    const w = (d.value / max) * chartW;
    ctx.fillStyle = "#8a5cf6"; ctx.fillRect(left, y, w, barH);
    ctx.fillStyle = "#fff"; ctx.font = "700 24px Inter"; ctx.textBaseline = "middle";
    ctx.textAlign = "right"; ctx.fillText(d.name, left - 16, y + barH/2);
    ctx.textAlign = "left";
    ctx.fillText(Math.round(d.value).toLocaleString(), left + w + 12, y + barH/2);
  });
}
// browser: requestAnimationFrame loop incrementing `frame`
// offline render: loop frame=0..N, drawFrame, then pipe canvas.toBuffer() to ffmpeg
```

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Bars flicker / teleport | Position from array index, or a CSS/D3 transition fighting the frame | Derive y from interpolated rank; remove all library timers |
| Numbers jitter in width | Proportional figures | `font-variant-numeric: tabular-nums` |
| Race feels like a slideshow | Playing one raw row per second | Interpolate continuously between rows (`f`) |
| Last frame overshoots | No clamp on the final segment | Clamp `f` to `[0,1]` and `i` to `rows.length-2` |
| Can't track the leader | Too fast | Raise `SECONDS_PER_ROW`; add a slow-down on key crossovers |
