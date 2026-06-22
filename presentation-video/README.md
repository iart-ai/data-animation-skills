# Presentation Video

Turn a deck or slide outline into a narrated, auto-advancing animated video — each slide rebuilt as motion graphics with staggered build reveals and timing locked to the voiceover, not a screen-recording of someone clicking through slides. For analysts, PMs, and data storytellers who want a self-playing deck where reveals land on the words being spoken.

## When it activates

- "Make an animated presentation" or "turn a pitch deck into a video."
- "Convert slides / a deck to video" or "build a deck video."
- "Narrate a slide deck as a video" or "auto-advance slides synced to a voiceover."

## Example prompts

- "Turn this pitch deck into a narrated video where each slide lasts as long as its voiceover."
- "Convert this bullet outline into animated slides with sequential build reveals."
- "Make a deck video with quiet fade transitions and a pronounced one on section dividers."

## What's inside

- `SKILL.md` — narration-owns-the-clock rule (audio length → slide duration), build reveals vs. decoration, `<TransitionSeries>` sequencing, title/section dividers, legibility and brand theming, and the outline → script → render workflow.
- `references/slide-components.md` — runnable Remotion `Slide`, staggered `BulletList`, animated `TitleCard`, and `SectionDivider` with a shared theme and per-slide narration mounted via `<Audio>`.
- `references/sequencing.md` — the full deck sequencer: `calculateMetadata` that measures every narration clip, `<TransitionSeries>` assembly, transition frame-budget accounting, and a JSON deck-schema → video data flow.

---
Part of **[Data Video Skills](../)** · Built by **[iart.ai](https://iart.ai)** — the AI motion agent for editable, on-brand motion graphics.
