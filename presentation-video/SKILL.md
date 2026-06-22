---
name: presentation-video
description: This skill should be used when the user asks to "make an animated presentation", "turn a pitch deck into a video", "convert slides/a deck to video", "make a presentation video", "build a deck video", "narrate a slide deck as a video", or "auto-advance slides synced to a voiceover". Covers rebuilding slides as motion graphics with staggered build reveals, slide-to-slide transitions, and timing locked to per-slide narration.
version: 0.1.0
---

# Presentation Video

Turn a deck or slide outline into a narrated, auto-advancing animated video. Each slide is **rebuilt as motion graphics** — staggered element reveals, real transitions, timing locked to the voiceover — not a screen-recording of someone clicking through PowerPoint.

## When to use

- A pitch deck, webinar, or course module that should become a self-playing video.
- A bullet-point outline that needs to become narrated, animated slides.
- A deck where slide timing must track a voiceover exactly (auto-advance, no clicking).

## Rebuild, don't record

A screen-recording inherits the deck's static look, mouse jitter, and abrupt slide cuts. Rebuilding in code (Remotion) buys: per-frame-deterministic reveals, transitions that actually ease, text that stays crisp at any resolution, and slide durations driven by audio length. The deck becomes the *script*; the video is a new artifact.

## The one rule: narration owns the clock

**Each slide lasts exactly as long as its narration audio, plus a small handle.** Measure every voiceover clip's real duration, convert to frames, and make that the slide's length. Never eyeball "this slide feels like ~5s" — it always drifts and the audio gets cut off or runs dry.

```ts
import { getAudioDurationInSeconds } from "@remotion/media-utils";
// In a Remotion calculateMetadata() / build step, per slide:
const sec = await getAudioDurationInSeconds(slide.audioUrl);
const HANDLE = 0.4; // breathing room after the voice ends
slide.durationInFrames = Math.ceil((sec + HANDLE) * fps);
```

Then the *reveals inside* the slide are scheduled against that duration — bullets land on the words, not on a fixed grid.

## Build reveals: animate to reveal, never to decorate

The point of a build is pacing: show each element when the narration reaches it, so the viewer reads the line being spoken — not three lines ahead. Stagger elements in; keep titles, logos, and background shapes static.

| Choice | Recommendation | Why |
|---|---|---|
| Enter motion | fade + 12–24px rise | Polished, calm; reads as "arriving" |
| Per-element duration | 0.25–0.5s | Fast enough to flow, slow enough to register |
| Stagger between bullets | 0.15–0.4s | Sequential, not a wall of text |
| Avoid | fly-in, spin, bounce, typewriter | Reads as decoration; competes with the voice |

Drive the stagger off `useCurrentFrame()` so it stays in lockstep with the rendered audio (see `references/slide-components.md` for the full component).

```tsx
import { useCurrentFrame, interpolate, spring, useVideoConfig } from "remotion";

const Bullet: React.FC<{ index: number; children: React.ReactNode }> = ({ index, children }) => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const STAGGER = 8;                         // frames between bullets (~0.27s @ 30fps)
  const enter = frame - index * STAGGER;     // each bullet starts later
  const p = spring({ frame: enter, fps, config: { damping: 200 }, durationInFrames: 12 });
  return (
    <li style={{ opacity: p, transform: `translateY(${interpolate(p, [0, 1], [18, 0])}px)` }}>
      {children}
    </li>
  );
};
```

## Slide-to-slide transitions

Sequence slides with `<TransitionSeries>` from `@remotion/transitions`. Each slide is a `Sequence` with its narration-derived `durationInFrames`; a `Transition` sits between them. Keep transitions quiet (short fade or gentle slide) — the content is the star, and a busy wipe on every cut gets tiring fast.

```tsx
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";

<TransitionSeries>
  {slides.map((s, i) => (
    <React.Fragment key={s.id}>
      <TransitionSeries.Sequence durationInFrames={s.durationInFrames}>
        <Slide {...s} />
      </TransitionSeries.Sequence>
      {i < slides.length - 1 && (
        <TransitionSeries.Transition
          timing={linearTiming({ durationInFrames: 15 })}
          presentation={fade()}
        />
      )}
    </React.Fragment>
  ))}
</TransitionSeries>
```

Reserve a bigger transition (a slide/push) for **section dividers** only — it signals "new topic" and earns the extra motion. A transition borrows frames from both neighbors, so add its length back into the slide budget if a slide's narration must play in full. See `references/sequencing.md`.

## Title & section dividers

A title card and section dividers give the video rhythm and let the viewer breathe between dense slides. Treat a divider as its own short "slide": a large section number/word, the section title, a hold of ~1.5–2s, and a more pronounced transition on either side. Keep the divider's visual language (color, type) consistent so it reads as punctuation, not a different video.

## Keep text legible & on-brand

Slides are text-dense, so legibility is the whole game.

- **Type scale, not deck scale.** A deck viewed on a laptop ≠ a video viewed in a feed. Bump body to ≥ 28–32px at 1080p; titles 56–80px. If a slide needs a 16px footnote to fit, it needs to be split into two slides.
- **One idea per slide.** If narration for a slide runs long, split it — never shrink type to fit.
- **Contrast + safe margins.** Hold a ≥ 4.5:1 contrast ratio; keep all content within the center 90% so it survives platform cropping.
- **Lock the brand in a theme object.** Colors, fonts, logo, and spacing live in one `theme` so every slide is consistent and a rebrand is a one-line change.

```ts
export const theme = {
  bg: "#0B0B12", fg: "#F5F5F7", accent: "#6C5CE7",
  font: "Inter, system-ui, sans-serif",
  titlePx: 72, bodyPx: 30, maxWidthPct: 90,
} as const;
```

## Workflow

1. **Outline → script.** One slide = one idea + the exact narration line(s). This is the source of truth.
2. **Generate/record narration per slide.** One audio file per slide (TTS or recorded).
3. **Measure audio → set slide durations.** `getAudioDurationInSeconds` → frames (the rule above).
4. **Build slide components** with staggered reveals tied to `useCurrentFrame()`.
5. **Sequence** with `<TransitionSeries>`; add title + section dividers.
6. **Mount each slide's audio** inside its `Sequence` so voice and visuals can never drift.
7. **Legibility/brand pass**, then render to MP4.

## Output checklist

- Every slide's `durationInFrames` is derived from its narration audio, not guessed.
- Each slide's audio is mounted inside its own `Sequence` (visuals + voice locked together).
- Bullets reveal sequentially via `useCurrentFrame()`; titles/logos static.
- Reveals are fade+rise, 0.25–0.5s, no decorative effects.
- Slide-to-slide transitions are quiet; bigger transitions only on section dividers.
- Body type ≥ 28px @ 1080p, ≥ 4.5:1 contrast, content within center 90%.
- One idea per slide; long narration splits the slide instead of shrinking type.

## Reference files

- `references/slide-components.md` — complete runnable Remotion `Slide`, staggered `BulletList`, animated `TitleCard`, and `SectionDivider` components with a shared theme and per-slide narration mounted via `<Audio>`.
- `references/sequencing.md` — the full deck sequencer: `calculateMetadata` that measures every narration clip and sets total + per-slide durations, `<TransitionSeries>` assembly, transition frame-budget accounting, and a JSON deck-schema → video data flow.
