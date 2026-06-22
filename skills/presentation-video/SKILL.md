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

## Narrative flow: the deck is one argument, not N slides

Before timing or motion, fix the *story*. A presentation video that advances itself has no presenter to recover a lost thread — the structure must carry the logic on its own. Build the spine first; slides are beats on it.

- **One throughline (the spine).** State the deck's single takeaway in one sentence. Every slide must advance that sentence; if a slide doesn't, cut it or move it to an appendix. The viewer should be able to recount the spine after one watch.
- **One message per slide.** A slide makes exactly one point — its title *is* that point, phrased as a claim ("Churn is our real growth tax"), not a label ("Churn"). Two ideas means two slides.
- **Transitions carry logic, not just motion.** The cut between slides should answer "so what's next?" — therefore, but, which means, here's the proof. Match the motion to the logical move: a quiet fade for "and also", a bigger section push for "new act". A transition that doesn't mark a logical step is just decoration.
- **Open with a hook, close with a CTA.** The first slide earns attention in ~5s — a sharp question, a surprising stat, the stakes — not an agenda or a logo hold. The last slide states the single action you want ("Start the pilot", "Invest $2M for 15%"), held still as the takeaway frame.

### Tension → resolution (the pitch spine)

A pitch is a drama, not a feature list. Sequence it so tension rises, then resolves — each act sets up the next.

| Act | Beat | Job |
|---|---|---|
| Problem | "Here's what's broken / at stake" | Create tension the viewer feels |
| Insight | "Here's what everyone misses" | The non-obvious turn; earn the solution |
| Solution | "Here's our answer" | Resolve the tension the problem built |
| Proof | "Here's why it works" | Traction, demo, numbers |
| Ask | "Here's what we want" | One concrete CTA |

Don't reveal the solution before the problem has landed — an unearned solution feels like a feature, not a relief. Map your slides onto these acts; if an act is missing the argument has a hole.

### Pacing: keep the build in sync with the voice

The throughline plays out in time, so on-screen build and VO must advance together — narration owns the clock (next section), and reveals are scheduled against it so the viewer reads the line *as it's spoken*, never ahead. One message per slide makes this natural: a slide carries one point for exactly as long as the voice argues it, then the transition marks the logical step to the next. Long narration on one point means split the slide, not race the build.

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

## Deliver & verify (rendered stills → MP4)

> **Packaged helper** (`scripts/`): tile your stills with `scripts/contact-sheet.sh sheet.png f-hook.png f-mid.png f-end.png`, then assert the encode with `scripts/probe-mp4.sh out.mp4 [WxH] [fps]`. See `scripts/README.md`.

Remotion is frame-deterministic — every bullet build, transition, and slide position is a pure function of `useCurrentFrame()`, so you can render any exact frame headlessly with no seek harness. The deliverable is a narrated MP4, so still-inspection catches a clipped title, an off-brand color, or a bullet that overflows before you waste a full encode.

**Output contract:**
- A Remotion project with the deck registered (`<Composition>` + zod `schema` + `defaultProps`), all motion frame-driven (no timers / `Date.now()` / `Math.random()`); per-slide audio mounted inside its own `Sequence`.
- Deliverable = the rendered `out/*.mp4` (plus the project, so the user can re-render with new slides/narration).
- Slide durations are narration-derived — compute total + per-slide `durationInFrames` in `calculateMetadata` (it measures each audio clip), never by hand.

**Verify loop — render stills → inspect → encode.** Render single frames first (cheap, no video encode), then encode only once each slide builds and reads right.

```bash
# 1. Frame-exact stills — with the SHIPPED props. Land one inside each slide's settled state:
npx remotion still Deck out/f-title.png --frame=20  --props='{"slides":[...]}'   # title card settled
npx remotion still Deck out/f-mid.png   --frame=240 --props='{"slides":[...]}'   # a content slide, bullets built
npx remotion still Deck out/f-end.png   --frame=899 --props='{"slides":[...]}'   # last frame = durationInFrames - 1

# 2. Inspect each PNG — FIDELITY (slide text exact, numbers/labels correct, brand colors/fonts applied)
#    AND artifacts (text overflow, body type too small, content outside center 90% safe-crop, missing
#    font, low contrast, a bullet caught mid-build where the slide should be settled).

# 3. Only after the stills check out, encode:
npx remotion render Deck out/deck.mp4 --props='{"slides":[...]}'
```

- Use `npx remotion compositions` to read the deck's `durationInFrames`/`fps` and pick the end frame; sample a still **inside each slide's hold**, not on a transition, to judge the built state.
- **Data-driven / batch**: verify ONE representative deck (real slides + narration) via stills *before* rendering every variant — catch a type-scale or overflow bug once, not per deck.
- **README demo GIF for free**: `npx remotion render Deck out/demo.gif --codec=gif`.

**Before you finish:**
1. `npx remotion still` renders cleanly at title, a content slide, and the last frame — no errors, no missing fonts.
2. Slide text is exact, body type ≥ 28px @ 1080p, ≥ 4.5:1 contrast, all content inside the center 90% at each frame.
3. Frame-driven only — no `Date.now()` / `Math.random()` / library timers (determinism holds in CI).
4. The **shipped** props render correctly (not just `defaultProps`) — right slides, narration mounted, durations from audio.
5. Full MP4 encoded and plays (voice + visuals locked); (optional) GIF rendered for the README.

## Reference files

- `references/slide-components.md` — complete runnable Remotion `Slide`, staggered `BulletList`, animated `TitleCard`, and `SectionDivider` components with a shared theme and per-slide narration mounted via `<Audio>`.
- `references/sequencing.md` — the full deck sequencer: `calculateMetadata` that measures every narration clip and sets total + per-slide durations, `<TransitionSeries>` assembly, transition frame-budget accounting, and a JSON deck-schema → video data flow.
