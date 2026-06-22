---
name: animated-infographic
description: This skill should be used when the user asks to "make an animated infographic", "create an infographic video", "build an infographic animation", "animate an explainer infographic", "do a sequenced reveal of icons and stats", "animate icons with key numbers and connectors", or "lay out a designed infographic that animates in sequence". Covers infographic layout & visual hierarchy, staggered element reveals, animating icons/pictograms, key-number counters, connectors/flow, and section pacing. For heavy charting (bar chart race, charts from a CSV) use the chart-animation skill instead.
version: 0.1.0
---

# Animated Infographic

Build a *designed* infographic — icons, a few key numbers, short text, and simple shapes arranged with real visual hierarchy — then animate it in sequence so each element earns its moment. The craft is composition first, choreography second: a static layout that reads on its own, revealed in a staggered cascade that guides the eye exactly where you want it.

## When to use

- Mixed-layout explainers: "X in 3 steps", "how it works", "by the numbers", a stat-card grid.
- Sequenced reveal of icon + stat + label + connector as a single designed scene.
- Pictogram/icon animation, count-up key numbers, flow connectors between steps.

This skill owns the **composed infographic**. For a chart that *is* the content (bar chart race, animated line/area, a graph driven by a CSV), use **chart-animation** — and compose its chart as one block inside an infographic here rather than rebuilding it.

## The one rule: design the static frame first

A great animated infographic is a great *static* infographic that happens to move. Lay out and balance the full composition — every icon, number, and label in its final position — before adding a single keyframe. Animation then only controls *when* each already-placed element appears; it never decides layout. This prevents the most common failure: elements that fly to positions that were never designed, so the final held frame looks accidental.

Scope tightly: one central insight, **3–5 supporting data points**, in 30–90s. More than five competing elements and the cascade reads as chaos.

## Visual hierarchy → reveal order

Reveal order *is* the hierarchy. The eye follows appearance, so animate elements in importance order, not layout order. Map each element to a tier, then reveal tier by tier.

| Tier | Element | Reveal | Motion |
|---|---|---|---|
| 1 | Section title / central insight | first, alone | fade + slight rise |
| 2 | Icon (anchors each item) | per item, leads its group | pop / scale-overshoot |
| 3 | Key number (counter) | right after its icon | count-up, ease-out |
| 4 | Label / caption | settles under the number | fade, no motion drama |
| 5 | Connector / flow line | links items as they complete | draw-on (path length) |

Use size, weight, and color for static hierarchy; use **timing and motion** for the animated layer. One loud thing at a time — never count two numbers at once.

## Stagger: the cascade

Reveal sibling items with a fixed delay between them (a *stagger*) so the group reads as a sequence, not a flash. Drive every element from the current frame as a pure function — never wall-clock time or a library's animation loop — so renders are deterministic.

```jsx
import { useCurrentFrame, spring, useVideoConfig } from "remotion";

const STAGGER = 8; // frames between siblings (~0.27s @30fps)

function Item({ index, children }) {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const local = frame - index * STAGGER;            // this item's own clock
  const enter = spring({ frame: local, fps, config: { damping: 14, mass: 0.6 } });
  return (
    <div style={{ opacity: enter, transform: `translateY(${(1 - enter) * 24}px)` }}>
      {children}
    </div>
  );
}
```

A stagger of 6–10 frames feels crisp; above ~15 it drags. Keep one enter curve across all siblings so the cascade reads as confidence, not noise.

## Animating icons: the pop

Icons anchor each item, so give them the strongest entrance — a scale-overshoot ("pop") that settles. Use a spring with low damping for the bounce; never linear scale (reads mechanical).

```jsx
const pop = spring({ frame: local, fps, config: { damping: 10, stiffness: 180, mass: 0.5 } });
// pop overshoots past 1 then settles → lively
<g transform={`scale(${pop})`} style={{ transformOrigin: "center" }}>{icon}</g>
```

For pictogram fills (e.g. "7 of 10 people"), reveal units on the same stagger and clip the partial unit with a mask rather than scaling it.

## Key numbers: counters that settle

Interpolate the underlying number, ease it, then format on render. Two musts: **round before formatting**, and use `tabular-nums` so the layout doesn't jitter as digits change. (Shared craft with chart-animation — see its counter section for currency/percent variants.)

```jsx
import { interpolate, Easing } from "remotion";

const raw = interpolate(frame - delay, [0, 30], [0, 1287], {
  extrapolateLeft: "clamp", extrapolateRight: "clamp",
  easing: Easing.out(Easing.cubic),   // "settling on a number"
});
const label = new Intl.NumberFormat("en-US").format(Math.round(raw)); // "1,287"
// CSS: font-variant-numeric: tabular-nums; line-height: 1;
```

Ease-out on a headline number reads as arriving; linear reads as a robotic odometer. Hold each number ~0.5s before the next item begins.

## Connectors & flow

Connectors turn a grid of cards into a *process*. Animate a line/arrow by drawing its path on, synced to land just as the item it points to finishes — motion that says "now look here next."

```jsx
const pathRef = useRef();
const len = pathRef.current?.getTotalLength?.() ?? 0;
const draw = interpolate(frame - delay, [0, 18], [0, 1], { extrapolateRight: "clamp" });
<path ref={pathRef} d="M40,0 C40,30 40,30 40,60" fill="none" stroke="#444" strokeWidth={3}
  strokeDasharray={len} strokeDashoffset={len * (1 - draw)} />
```

Draw the connector *between* the two items it joins, in the gap after the source appears and before the target — the line leads the eye into the next reveal.

## Section pacing

Budget time per element, not per second of polish. A viewer can only track one moving thing at a time.

| Beat | Budget |
|---|---|
| Title hold (let it land) | 0.8–1.2s |
| Per item (icon pop → counter → label) | 1.2–2.0s |
| Connector draw | 0.4–0.6s, overlaps into next item |
| Final composed hold (screenshot-able) | 2–3s |

End on the **complete static infographic held still** for 2–3s — the takeaway is the assembled frame, so let it settle with no motion competing.

## Output checklist

- Static layout designed and balanced *before* any keyframes; held final frame looks intentional.
- One central insight, 3–5 data points, 30–90s.
- Reveal order follows hierarchy (title → icon → number → label → connector), not layout order.
- Siblings cascade on a consistent 6–10 frame stagger with one enter curve.
- Icons pop with a spring overshoot; numbers count up eased + `tabular-nums`; only one number animates at a time.
- Connectors draw on to lead the eye into the next reveal.
- Every value is a pure function of `useCurrentFrame()`; no library timers.
- Final composed frame holds ≥2s.

## Deliver & verify (rendered stills → MP4)

> **Packaged helper** (`scripts/`): tile your stills with `scripts/contact-sheet.sh sheet.png f-hook.png f-mid.png f-end.png`, then assert the encode with `scripts/probe-mp4.sh out.mp4 [WxH] [fps]`. See `scripts/README.md`.

Remotion is frame-deterministic — every icon pop, counter value, and connector draw is a pure function of `useCurrentFrame()`, so you can render any exact frame headlessly with no seek harness. The infographic carries **key numbers**, so still-inspection catches a wrong stat or a stagger that lands off-canvas before you waste an encode.

**Output contract:**
- A Remotion project with the scene registered (`<Composition>` + zod `schema` + `defaultProps`), all motion frame-driven (no timers / `Date.now()` / `Math.random()`).
- Deliverable = the rendered `out/*.mp4` (plus the project, so the user can re-render with new stats/icons).
- Duration data-dependent (N items × per-item budget)? compute it in `calculateMetadata`, not by hand.

**Verify loop — render stills → inspect → encode.** Render single frames first (cheap, no video encode), then encode only once the layout and numbers are right.

```bash
# 1. Frame-exact stills — with the SHIPPED props. Pick frames that catch each tier:
npx remotion still Infographic out/f-title.png --frame=20  --props='{"items":[...]}'  # title + first pop
npx remotion still Infographic out/f-mid.png   --frame=90  --props='{"items":[...]}'  # mid cascade
npx remotion still Infographic out/f-end.png   --frame=149 --props='{"items":[...]}'  # final composed hold (= durationInFrames - 1)

# 2. Inspect each PNG — FIDELITY (every key number exact, labels/captions correct, icons the right glyph)
#    AND artifacts (text overflow, icon off-canvas, clipped safe-area, missing font, connector pointing
#    at the wrong item, half-revealed element where a tier should be settled).

# 3. Only after the stills check out, encode:
npx remotion render Infographic out/infographic.mp4 --props='{"items":[...]}'
```

- Use `npx remotion compositions` to read `durationInFrames`/`fps`; the **final composed hold** is the money frame — verify it looks intentional, not mid-cascade.
- **Data-driven / batch**: verify ONE representative props set (stats + labels) via stills *before* batch-rendering all variants — catch a counter or layout bug once, not N times.
- **README demo GIF for free**: `npx remotion render Infographic out/demo.gif --codec=gif`.

**Before you finish:**
1. `npx remotion still` renders cleanly at title, mid-cascade, and final hold — no errors, no missing fonts/icons.
2. Every key number is **exact** (rounded, `tabular-nums`) and every element is inside the safe area at each frame.
3. Frame-driven only — no `Date.now()` / `Math.random()` / library timers (determinism holds in CI).
4. The **shipped** props render correctly (not just `defaultProps`) — right stats, right icons, right labels.
5. Full MP4 encoded and plays; (optional) GIF rendered for the README.

## Reference files

- `references/sequenced-infographic.md` — a complete runnable Remotion scene: a 3-step "how it works" infographic with staggered icon pops, eased count-up stats, draw-on connectors, a master timing map, and the make-it-data-driven (template × data) prop pattern. Includes a dependency-free inline-SVG variant for non-Remotion use.
