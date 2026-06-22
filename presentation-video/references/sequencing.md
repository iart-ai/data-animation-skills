# Sequencing: deck schema → narration-timed video

How to assemble many slides into one auto-advancing video where **every slide lasts exactly as long as its narration**. Covers the deck JSON schema, measuring audio in `calculateMetadata`, building total + per-slide durations, the `<TransitionSeries>` assembly, transition frame-budget accounting, and the batch render.

## 1. Deck schema (the source of truth)

Author the deck as data, not as a hardcoded component. One slide = one idea + the narration line + the path to its rendered audio. This is what you'd export from an outline or a deck-to-script step.

```ts
// deck.ts
export type DeckSlide =
  | { kind: "title"; id: string; title: string; subtitle?: string; audioUrl: string }
  | { kind: "section"; id: string; index: number; title: string; audioUrl: string }
  | { kind: "content"; id: string; title: string; bullets: string[]; audioUrl: string };

export type Deck = { fps: number; width: number; height: number; slides: DeckSlide[] };
```

```json
// deck.json — narration lines live next to the slide content
{
  "fps": 30, "width": 1920, "height": 1080,
  "slides": [
    { "kind": "title", "id": "t", "title": "Acme Q3", "subtitle": "Growth review", "audioUrl": "audio/t.mp3" },
    { "kind": "section", "id": "s1", "index": 1, "title": "Where we are", "audioUrl": "audio/s1.mp3" },
    { "kind": "content", "id": "c1", "title": "Three signals", "bullets": ["Revenue up 28%", "Churn down to 1.9%", "NPS at 61"], "audioUrl": "audio/c1.mp3" }
  ]
}
```

## 2. Measure narration → durations (the rule, in code)

`calculateMetadata` runs before render, so it is the right place to read each audio file's real length and turn it into frames. The composition's total duration is the sum; each slide carries its own frame count downstream as a prop.

```tsx
// Root.tsx
import { Composition, getInputProps } from "remotion";
import { getAudioDurationInSeconds } from "@remotion/media-utils";
import { DeckVideo } from "./DeckVideo";
import deck from "./deck.json";

const HANDLE = 0.4; // seconds of breathing room after each voice line

export const RemotionRoot: React.FC = () => (
  <Composition
    id="Deck"
    component={DeckVideo}
    fps={deck.fps}
    width={deck.width}
    height={deck.height}
    defaultProps={{ deck }}
    calculateMetadata={async ({ props }) => {
      const fps = props.deck.fps;
      // measure every narration clip in parallel
      const durations = await Promise.all(
        props.deck.slides.map((s) => getAudioDurationInSeconds(s.audioUrl))
      );
      const timed = props.deck.slides.map((s, i) => ({
        ...s,
        durationInFrames: Math.ceil((durations[i] + HANDLE) * fps),
      }));
      const total = timed.reduce((n, s) => n + s.durationInFrames, 0);
      return { durationInFrames: total, props: { deck: { ...props.deck, slides: timed } } };
    }}
  />
);
```

Now each slide object has a `durationInFrames` that is *guaranteed* to fit its voiceover. No slide can cut off the audio, and none holds an awkward silence beyond `HANDLE`.

## 3. Transition frame-budget accounting

A `<TransitionSeries.Transition>` does **not** add time — it overlaps its neighbors, consuming frames from the end of the outgoing slide and the start of the incoming one. So a 15-frame fade between slide A and B steals ~7–8 frames of visible time from each.

That is fine for the quiet fades between content slides (a few frames at the head/tail, where nothing is being read). But if a slide's narration must play *in full* with no overlap, add the transition length back into that slide's budget:

```ts
const TRANSITION = 15;                 // frames
const SECTION_TRANSITION = 24;         // bigger, for dividers
// give a slide extra frames so the overlap doesn't eat into spoken words
slide.durationInFrames += TRANSITION;  // one transition touches it (interior slides: + both, see assembly)
```

Rule of thumb: keep transitions short (10–18 frames) so the stolen time lands in the slide's lead-in/tail, not over a spoken sentence. Reserve `SECTION_TRANSITION` (a slide/push) for dividers only.

## 4. Assembly with `<TransitionSeries>`

Map the timed deck into sequences, choosing the component by `kind` and the transition by neighbor type — a bigger `slide()` around section dividers, a quiet `fade()` everywhere else.

```tsx
// DeckVideo.tsx
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";
import { slide } from "@remotion/transitions/slide";
import { Slide } from "./Slide";
import { TitleCard } from "./TitleCard";
import { SectionDivider } from "./SectionDivider";
import type { Deck } from "./deck";

const render = (s: any) => {
  if (s.kind === "title") return <TitleCard title={s.title} subtitle={s.subtitle} />;
  if (s.kind === "section") return <SectionDivider index={s.index} title={s.title} />;
  return <Slide {...s} />;
};

export const DeckVideo: React.FC<{ deck: Deck }> = ({ deck }) => {
  const slides = deck.slides as any[];
  return (
    <TransitionSeries>
      {slides.map((s, i) => {
        const next = slides[i + 1];
        const big = s.kind === "section" || next?.kind === "section";
        return (
          <React.Fragment key={s.id}>
            <TransitionSeries.Sequence durationInFrames={s.durationInFrames}>
              {render(s)}
            </TransitionSeries.Sequence>
            {next && (
              <TransitionSeries.Transition
                timing={linearTiming({ durationInFrames: big ? 24 : 15 })}
                presentation={big ? slide() : fade()}
              />
            )}
          </React.Fragment>
        );
      })}
    </TransitionSeries>
  );
};
```

Each slide's `<Audio>` is mounted *inside* its own component (see `slide-components.md`), so the sequencer positioning the visuals also positions the voice — they cannot desync.

## 5. Render

```bash
# single deck
npx remotion render Deck out/deck.mp4

# many decks from one template (e.g. one per customer/region)
for f in decks/*.json; do
  name=$(basename "$f" .json)
  npx remotion render Deck "out/$name.mp4" --props="$f"
done
```

## Pitfalls

- **Guessed durations.** Any hardcoded slide length will drift from the voice. Always measure (step 2).
- **Audio mounted at the deck level.** Mount narration *per slide*, inside the slide component, or a re-order silently misaligns every voice line.
- **Heavy transition on every cut.** Tiring, and it eats spoken time. Quiet fades for content; big transitions only at dividers.
- **One mega-slide.** If narration for a slide is long and bullets overflow, split into two slides — never shrink the type to fit.
- **Missing fonts at render.** Load brand fonts with `@remotion/google-fonts` or `staticFile`, or text reflows between preview and render.
