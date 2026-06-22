# Sequenced Infographic — runnable reference

A complete, runnable Remotion scene: a 3-step "How it works" infographic where each step reveals as an **icon pop → eased count-up stat → label**, joined by **draw-on connectors**, ending on a held static frame. Below the Remotion version is a master timing map, a template×data pattern, and a dependency-free inline-SVG variant.

## Mental model

Three clocks, one source:

1. **Frame** — `useCurrentFrame()` is the only clock. Every value is a pure function of it.
2. **Item-local frame** — `frame - itemStartFrame` gives each item its own 0-based timeline, so the same `Step` component animates correctly wherever it sits in the sequence.
3. **Element offset** — within an item, the icon leads (offset 0), the counter follows (offset ~6f), the label trails (offset ~12f). Hierarchy becomes timing.

Design the static layout first (positions below are final), then these clocks only decide *when* each placed element fades in.

## Remotion: `SequencedInfographic.tsx`

```tsx
import {
  AbsoluteFill, useCurrentFrame, useVideoConfig, spring, interpolate, Easing, Sequence,
} from "remotion";
import { useRef } from "react";

// ---- Data is a PROP, never hardcoded (enables template × data, see below) ----
export type Step = { icon: "search" | "wrench" | "rocket"; value: number; suffix?: string; label: string };
export type Props = { title: string; steps: Step[]; theme?: Partial<Theme> };

type Theme = { bg: string; ink: string; sub: string; accent: string; line: string; font: string };
const DEFAULT_THEME: Theme = {
  bg: "#0E1116", ink: "#F5F7FA", sub: "#9AA4B2", accent: "#5B8DEF", line: "#3A4250",
  font: "Inter, system-ui, sans-serif",
};

// Timing constants (frames @ 30fps) — the choreography lives here
const TITLE_HOLD = 30;     // 1.0s before steps begin
const STEP_STRIDE = 48;    // 1.6s per step (pop → count → label, then connector overlaps)
const ICON_OFFSET = 0;
const COUNT_OFFSET = 6;
const LABEL_OFFSET = 14;
const CONNECTOR_OFFSET = 30; // draws after the source step, leading into the next

// ---------- Count-up number that settles (ease-out cubic) ----------
function Counter({ from = 0, to, suffix = "", startFrame }: { from?: number; to: number; suffix?: string; startFrame: number }) {
  const frame = useCurrentFrame();
  const raw = interpolate(frame - startFrame, [0, 30], [from, to], {
    extrapolateLeft: "clamp", extrapolateRight: "clamp", easing: Easing.out(Easing.cubic),
  });
  const n = Math.round(raw);
  const text = `${new Intl.NumberFormat("en-US").format(n)}${suffix}`;
  return <span style={{ fontVariantNumeric: "tabular-nums", lineHeight: 1 }}>{text}</span>;
}

// ---------- Minimal inline icon set (replace with your own SVG paths) ----------
function Icon({ name, color }: { name: Step["icon"]; color: string }) {
  const paths: Record<Step["icon"], string> = {
    search: "M11 19a8 8 0 1 1 5.3-2L21 21",
    wrench: "M14 7a4 4 0 0 1-5 5L4 17l3 3 5-5a4 4 0 0 1 5-5l-3-3z",
    rocket: "M5 15l4 4M9 11a8 8 0 0 1 9-6 8 8 0 0 1-6 9l-3 3-3-3 3-3z",
  };
  return (
    <svg width={56} height={56} viewBox="0 0 24 24" fill="none"
      stroke={color} strokeWidth={2} strokeLinecap="round" strokeLinejoin="round">
      <path d={paths[name]} />
    </svg>
  );
}

// ---------- One step: icon pops, number counts, label fades — all on item-local frame ----------
function StepCard({ step, startFrame, theme }: { step: Step; startFrame: number; theme: Theme }) {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const local = frame - startFrame;

  // Icon: spring with overshoot ("pop")
  const pop = spring({ frame: local - ICON_OFFSET, fps, config: { damping: 10, stiffness: 180, mass: 0.5 } });

  // Label: plain fade + rise (no drama — it's tier 4)
  const labelIn = spring({ frame: local - LABEL_OFFSET, fps, config: { damping: 16, mass: 0.6 } });

  return (
    <div style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 12, width: 240 }}>
      <div style={{
        width: 96, height: 96, borderRadius: 24, display: "grid", placeItems: "center",
        background: "rgba(91,141,239,0.12)", transform: `scale(${pop})`, transformOrigin: "center",
      }}>
        <Icon name={step.icon} color={theme.accent} />
      </div>
      <div style={{ color: theme.ink, fontSize: 52, fontWeight: 800 }}>
        <Counter to={step.value} suffix={step.suffix} startFrame={startFrame + COUNT_OFFSET} />
      </div>
      <div style={{
        color: theme.sub, fontSize: 20, textAlign: "center",
        opacity: labelIn, transform: `translateY(${(1 - labelIn) * 12}px)`,
      }}>
        {step.label}
      </div>
    </div>
  );
}

// ---------- Connector: a horizontal line that draws on between two steps ----------
function Connector({ startFrame }: { startFrame: number }) {
  const frame = useCurrentFrame();
  const ref = useRef<SVGPathElement>(null);
  const len = ref.current?.getTotalLength?.() ?? 120;
  const draw = interpolate(frame - startFrame, [0, 18], [0, 1], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
  return (
    <svg width={120} height={48} viewBox="0 0 120 48" style={{ overflow: "visible" }}>
      <path ref={ref} d="M8 24 H112" fill="none" stroke="#3A4250" strokeWidth={3} strokeLinecap="round"
        strokeDasharray={len} strokeDashoffset={len * (1 - draw)} />
      <path d="M104 18 L114 24 L104 30" fill="none" stroke="#3A4250" strokeWidth={3}
        strokeLinecap="round" strokeLinejoin="round"
        style={{ opacity: interpolate(frame - startFrame, [14, 18], [0, 1], { extrapolateLeft: "clamp", extrapolateRight: "clamp" }) }} />
    </svg>
  );
}

export const SequencedInfographic: React.FC<Props> = ({ title, steps, theme: t }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const theme = { ...DEFAULT_THEME, ...t };

  const titleIn = spring({ frame, fps, config: { damping: 18, mass: 0.7 } });

  return (
    <AbsoluteFill style={{ background: theme.bg, fontFamily: theme.font, padding: 80,
      display: "flex", flexDirection: "column", justifyContent: "center", alignItems: "center", gap: 64 }}>
      <h1 style={{ color: theme.ink, fontSize: 56, fontWeight: 800, margin: 0, textAlign: "center",
        opacity: titleIn, transform: `translateY(${(1 - titleIn) * 20}px)` }}>
        {title}
      </h1>

      <div style={{ display: "flex", alignItems: "flex-start", gap: 8 }}>
        {steps.map((step, i) => {
          const start = TITLE_HOLD + i * STEP_STRIDE;
          return (
            <div key={`${step.label}-${i}`} style={{ display: "flex", alignItems: "center", gap: 8 }}>
              <StepCard step={step} startFrame={start} theme={theme} />
              {i < steps.length - 1 && (
                // connector draws ~1s after this step starts, leading into the next
                <div style={{ marginTop: 36 }}>
                  <Connector startFrame={start + CONNECTOR_OFFSET} />
                </div>
              )}
            </div>
          );
        })}
      </div>
    </AbsoluteFill>
  );
};
```

### Register it (`Root.tsx`)

```tsx
import { Composition } from "remotion";
import { SequencedInfographic } from "./SequencedInfographic";

const DEMO = {
  title: "How it works",
  steps: [
    { icon: "search", value: 3,    suffix: "k+", label: "Sources scanned" },
    { icon: "wrench", value: 92,   suffix: "%",  label: "Auto-matched" },
    { icon: "rocket", value: 1287, suffix: "",   label: "Reports shipped" },
  ],
} as const;

export const Root = () => (
  <Composition
    id="SequencedInfographic"
    component={SequencedInfographic}
    durationInFrames={TITLE_HOLD_PLUS_STEPS()} // e.g. 30 + 3*48 + 90 hold ≈ 264
    fps={30}
    width={1920}
    height={1080}
    defaultProps={DEMO}
  />
);
function TITLE_HOLD_PLUS_STEPS() { return 30 + 3 * 48 + 90; }
```

```bash
npm i remotion @remotion/cli react react-dom
npx remotion studio        # preview, scrub the timeline
npx remotion render SequencedInfographic out/infographic.mp4
```

## Master timing map (3 steps, 30fps)

| Frame | s | Event |
|---|---|---|
| 0–30 | 0.0–1.0 | Title springs in, holds |
| 30 | 1.0 | Step 1 icon pops |
| 36 | 1.2 | Step 1 counter starts (settles by ~66 / 2.2s) |
| 44 | 1.5 | Step 1 label fades up |
| 60 | 2.0 | Connector 1 draws → arrow into step 2 |
| 78 | 2.6 | Step 2 icon pops (stride 48f) |
| 126 | 4.2 | Step 3 icon pops |
| ~174 | 5.8 | All elements settled |
| 174–264 | 5.8–8.8 | **Held static infographic** (screenshot-able) |

Only one counter runs at a time (steps are 1.6s apart, a counter settles in ~1s) — the eye is never asked to read two moving numbers.

## Template × data — one design, many infographics

`steps` is already a prop, so the same composition renders any dataset. Keep theme/layout fixed; change only the data.

```bash
# render the same infographic template for every dataset in /data
for f in data/*.json; do
  name=$(basename "$f" .json)
  npx remotion render SequencedInfographic "out/$name.mp4" --props="$f"
done
```

```json
// data/q2-results.json — shape matches Props
{
  "title": "Q2 by the numbers",
  "steps": [
    { "icon": "search", "value": 48,   "suffix": "k",  "label": "New signups" },
    { "icon": "wrench", "value": 99,    "suffix": "%",  "label": "Uptime" },
    { "icon": "rocket", "value": 2400000,"suffix": "",  "label": "API calls/day" }
  ]
}
```

Validate before rendering: confirm `steps.length` is 3–5, every `value` is a finite number, and `icon` is one of the known keys — bad data should fail loudly, not render a broken frame.

## Dependency-free variant (inline SVG + Web Animations API)

For a landing page or a quick capture without Remotion. The same choreography — staggered icon pop, counter, connector draw — driven by `requestAnimationFrame` (fine for live playback; for frame-exact MP4 export prefer the Remotion version above).

```html
<div id="infographic" style="display:flex;gap:8px;font-family:Inter,sans-serif;background:#0E1116;padding:60px">
  <template id="step">
    <div class="card" style="opacity:0;width:220px;text-align:center">
      <div class="icon" style="display:inline-grid;place-items:center;width:96px;height:96px;border-radius:24px;
        background:rgba(91,141,239,.12);transform:scale(0)"></div>
      <div class="num" style="color:#F5F7FA;font-size:52px;font-weight:800;font-variant-numeric:tabular-nums">0</div>
      <div class="lbl" style="color:#9AA4B2;font-size:20px"></div>
    </div>
  </template>
</div>
<script>
const STEPS = [
  { icon:"🔍", value:3,    suffix:"k+", label:"Sources scanned" },
  { icon:"🔧", value:92,   suffix:"%",  label:"Auto-matched" },
  { icon:"🚀", value:1287, suffix:"",   label:"Reports shipped" },
];
const root = document.getElementById("infographic");
const tpl  = document.getElementById("step");
const easeOutCubic = t => 1 - Math.pow(1 - t, 3);

STEPS.forEach((s, i) => {
  const node = tpl.content.cloneNode(true);
  const card = node.querySelector(".card");
  card.querySelector(".icon").textContent = s.icon;
  card.querySelector(".lbl").textContent  = s.label;
  const numEl = card.querySelector(".num");
  root.appendChild(node);

  const delay = i * 480;                         // 480ms stagger between steps
  card.animate([{opacity:0},{opacity:1}], {duration:300, delay, fill:"forwards"});
  // icon pop with overshoot
  card.querySelector(".icon").animate(
    [{transform:"scale(0)"},{transform:"scale(1.15)"},{transform:"scale(1)"}],
    {duration:500, delay:delay+60, fill:"forwards", easing:"cubic-bezier(.34,1.56,.64,1)"});
  // counter (ease-out), starts after the icon
  const t0 = delay + 200, dur = 900;
  const tick = now => {
    const p = Math.min(1, (now - start - t0) / dur);
    if (p >= 0) numEl.textContent = Math.round(easeOutCubic(p) * s.value).toLocaleString() + s.suffix;
    if (p < 1) requestAnimationFrame(tick);
  };
  let start; requestAnimationFrame(n => { start = n; requestAnimationFrame(tick); });
});
</script>
```

Swap the emoji for inline `<svg>` icons in production, and add SVG `<path>` connectors with an animated `stroke-dashoffset` (same draw-on technique as the Remotion `Connector`) to join the cards.

---
## Built by the team behind iart.ai

This skill is part of an open motion-graphics collection from iart.ai — the AI motion agent that turns data, scripts, and designs into editable motion graphics (Remotion → MP4). If you'd rather not hand-build this, iart.ai can build data-driven animated infographics from a template — change the text/data and re-export. → https://iart.ai
