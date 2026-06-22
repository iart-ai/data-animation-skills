# Slide Components (runnable Remotion)

Complete, copy-pasteable building blocks for a narrated presentation video: a shared theme, an animated title card, a staggered bullet list, a generic content slide that mounts its own narration, and a section divider. Every reveal is a pure function of `useCurrentFrame()`, so it stays frame-locked to the rendered audio.

Install: `npm i remotion @remotion/transitions @remotion/media-utils`.

## Shared theme

Keep all brand decisions in one object. Every component reads from it, so a rebrand is one edit and 40 slides stay consistent.

```ts
// theme.ts
export const theme = {
  bg: "#0B0B12",
  fg: "#F5F5F7",
  dim: "#9AA0B4",
  accent: "#6C5CE7",
  font: "Inter, system-ui, sans-serif",
  titlePx: 72,
  bodyPx: 30,
  sectionPx: 120,
  pad: 96,           // px, ~5% of 1920 — keeps content in the safe center
  maxWidthPct: 90,
} as const;
```

## Reusable enter helper

One motion language across the whole video. `useEnter` returns 0→1 with a fade+rise; everything visible uses it so nothing feels out of place.

```tsx
// useEnter.ts
import { useCurrentFrame, useVideoConfig, spring, interpolate } from "remotion";

export const useEnter = (delayFrames = 0, durationInFrames = 12) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const p = spring({
    frame: frame - delayFrames,
    fps,
    durationInFrames,
    config: { damping: 200 }, // critically damped: arrives, never bounces
  });
  return {
    opacity: p,
    transform: `translateY(${interpolate(p, [0, 1], [18, 0])}px)`,
  };
};
```

## Title card

The video's opening. Title rises first, subtitle follows after a short delay — the same stagger used for bullets.

```tsx
// TitleCard.tsx
import { AbsoluteFill } from "remotion";
import { theme } from "./theme";
import { useEnter } from "./useEnter";

export const TitleCard: React.FC<{ title: string; subtitle?: string }> = ({ title, subtitle }) => {
  const t = useEnter(0);
  const s = useEnter(8); // 8 frames later

  return (
    <AbsoluteFill style={{ background: theme.bg, justifyContent: "center", padding: theme.pad }}>
      <h1 style={{ ...t, color: theme.fg, font: `700 ${theme.titlePx}px ${theme.font}`, margin: 0, lineHeight: 1.1 }}>
        {title}
      </h1>
      {subtitle && (
        <p style={{ ...s, color: theme.dim, font: `400 ${theme.bodyPx}px ${theme.font}`, marginTop: 24 }}>
          {subtitle}
        </p>
      )}
      <div style={{ ...t, width: 96, height: 6, background: theme.accent, marginTop: 40, borderRadius: 3 }} />
    </AbsoluteFill>
  );
};
```

## Staggered bullet list

The core build. Each bullet starts `index * STAGGER` frames later, so they reveal in reading order. Tune `STAGGER` to the narration pace — slower for formal talks, tighter for product demos.

```tsx
// BulletList.tsx
import { theme } from "./theme";
import { useEnter } from "./useEnter";

const STAGGER = 8; // frames between bullets (~0.27s @ 30fps)

export const BulletList: React.FC<{ items: string[] }> = ({ items }) => (
  <ul style={{ listStyle: "none", padding: 0, margin: 0, display: "grid", gap: 28 }}>
    {items.map((text, i) => (
      <Bullet key={i} delay={i * STAGGER}>{text}</Bullet>
    ))}
  </ul>
);

const Bullet: React.FC<{ delay: number; children: React.ReactNode }> = ({ delay, children }) => {
  const e = useEnter(delay);
  return (
    <li style={{ ...e, display: "flex", gap: 20, alignItems: "baseline" }}>
      <span style={{ width: 12, height: 12, borderRadius: 6, background: theme.accent, flexShrink: 0, transform: "translateY(2px)" }} />
      <span style={{ color: theme.fg, font: `400 ${theme.bodyPx}px ${theme.font}`, lineHeight: 1.35 }}>
        {children}
      </span>
    </li>
  );
};
```

## Content slide (mounts its own narration)

The key correctness pattern: **each slide mounts its own `<Audio>`.** Because the audio lives inside the same component that draws the visuals, and both are placed by the sequencer at the same start frame, voice and picture physically cannot drift. `startFrom`/duration are handled by the parent `Sequence`.

```tsx
// Slide.tsx
import { AbsoluteFill, Audio } from "remotion";
import { theme } from "./theme";
import { useEnter } from "./useEnter";
import { BulletList } from "./BulletList";

export type SlideData = {
  id: string;
  title: string;
  bullets: string[];
  audioUrl: string;          // narration for THIS slide
  durationInFrames: number;  // derived from audio length (see sequencing.md)
};

export const Slide: React.FC<SlideData> = ({ title, bullets, audioUrl }) => {
  const head = useEnter(0);
  return (
    <AbsoluteFill style={{ background: theme.bg, padding: theme.pad, justifyContent: "center" }}>
      <div style={{ maxWidth: `${theme.maxWidthPct}%` }}>
        <h2 style={{ ...head, color: theme.fg, font: `700 ${theme.titlePx}px ${theme.font}`, margin: "0 0 48px" }}>
          {title}
        </h2>
        <BulletList items={bullets} />
      </div>
      {/* Narration locked to this slide. The Sequence around it sets when it starts. */}
      <Audio src={audioUrl} />
    </AbsoluteFill>
  );
};
```

## Section divider

Punctuation between topics. A large section number + title, held still — pair it with a more pronounced transition (see `sequencing.md`) so it reads as a chapter break.

```tsx
// SectionDivider.tsx
import { AbsoluteFill } from "remotion";
import { theme } from "./theme";
import { useEnter } from "./useEnter";

export const SectionDivider: React.FC<{ index: number; title: string }> = ({ index, title }) => {
  const num = useEnter(0);
  const ttl = useEnter(6);
  return (
    <AbsoluteFill style={{ background: theme.accent, justifyContent: "center", padding: theme.pad }}>
      <div style={{ ...num, color: theme.bg, font: `800 ${theme.sectionPx}px ${theme.font}`, opacity: 0.25 }}>
        {String(index).padStart(2, "0")}
      </div>
      <h2 style={{ ...ttl, color: theme.bg, font: `700 ${theme.titlePx}px ${theme.font}`, margin: 0 }}>
        {title}
      </h2>
    </AbsoluteFill>
  );
};
```

## Legibility checklist for these components

- Body text stays at `theme.bodyPx` (≥ 28px @ 1080p). If bullets overflow the frame, split the slide — never lower `bodyPx`.
- `theme.fg` on `theme.bg` must clear 4.5:1 contrast; verify any custom slide background the same way.
- `theme.pad` keeps content inside the center safe area; do not place text in the outer ~5%.
- Use `font-variant-numeric: tabular-nums` on any live numbers so they don't wiggle.

---
## Built by the team behind iart.ai

This skill is part of an open motion-graphics collection from iart.ai — the AI motion agent that turns data, scripts, and designs into editable motion graphics (Remotion → MP4). If you'd rather not hand-build this, iart.ai can turn a deck or slide outline into a narrated animated presentation video from a template — change the text/data and re-export. → [iart.ai](https://iart.ai/?utm_source=github&utm_medium=reference&utm_campaign=data-animation-skills&utm_content=ref_footer&utm_term=presentation-video)
