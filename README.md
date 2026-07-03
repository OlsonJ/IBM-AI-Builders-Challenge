# DeckFlow Web

A browser-only, two-deck DJ mixer built on [Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API) and [Elementary Audio](https://www.elementary.audio/) (WASM). No backend, no install beyond `npm`. Load two tracks, mix with EQ and filters, adjust tempo, and crossfade — all in the browser.

This is a teaching variant of a desktop DJ app. The goal is a codebase readable end-to-end in an afternoon.

---

## Getting started

```bash
npm install
npm run dev
```

Open the URL, click **Load track** on each deck (audio requires a user gesture), then press **▶ Play**.

```bash
npm run build   # TypeScript typecheck + production bundle
```

---

## What's implemented

| Phase | Feature | Status |
|-------|---------|--------|
| P0 | WASM audio boots; test tone on user gesture | ✅ |
| P1 | One deck: load → decode → VFS → `el.table` playback; play/pause; click-to-seek; canvas waveform + live playhead | ✅ |
| P2 | Per-deck signal chain: 3-band EQ + DJ LPF/HPF filter + volume + level meter (true stereo) | ✅ |
| P3 | Two decks + equal-power crossfader + master volume; single combined graph | ✅ |
| P4a | Varispeed tempo knob per deck (0.5×–2.0×, `±N%` display, double-click reset) | ✅ |
| P4b | Manual loops (floored-mod wrap), in-mix cue point, waveform markers | ⬜ in progress |
| P5 | BPM detection (Web Worker), beat grid, tap tempo, SYNC | ⬜ planned |
| P6 | Session library panel + drag-to-deck | ⬜ planned |

---

## Stack

| | |
|---|---|
| **Build** | [Vite](https://vitejs.dev/) 5 |
| **UI** | React 18 + TypeScript (strict) |
| **Audio** | [`@elemaudio/core`](https://www.npmjs.com/package/@elemaudio/core) + [`@elemaudio/web-renderer`](https://www.npmjs.com/package/@elemaudio/web-renderer) 4.x |
| **No backend** | Session-only; nothing persists between reloads |

---

## Project layout

```
src/
  audio.ts          AudioContext + WebRenderer boot (50 ms event poll)
  track.ts          decodeAudioData → VFS (L/R mono entries) + peak analysis
  deck.ts           DeckState + buildDeckSignal (transport, EQ, filter, volume, meter)
  useDeck.ts        Per-deck hook: reducer state, transport, playhead/meter
  App.tsx           Two decks, combined render graph, crossfader, master volume
  components/
    Waveform.tsx    Cached canvas: peaks + playhead + zoom (scroll to zoom)
    DeckPanel.tsx   Deck lane: load button, waveform, transport controls
    DeckControls.tsx  Mixer strip: EQ knobs + filter knob + tempo knob + volume fader
    Knob.tsx        Rotary control (drag to adjust, double-click to reset)
    Fader.tsx       Vertical fader + live level meter
    Mixer.tsx       Crossfader + master volume (centre column)
```

---

## How the audio graph works

Each deck is a chain of Elementary nodes reading the decoded track buffer directly from the VFS. The transport phase is an integrating accumulator:

```
position = baseNorm + accum(increment, seekGen)
```

- **`increment`** — normalized progress per output sample: `0` when paused, `tempo / (frames − 1)` when playing. Scaling by `tempo` is the varispeed — pitch moves with tempo (no time-stretch).
- **`accum`** — integrates at audio rate; resets when `seekGen` changes (a seek).
- **`baseNorm`** — the normalized position of the last seek.

The mixer chain per deck: `lowshelf EQ → peak EQ → highshelf EQ → LPF/HPF filter → volume → meter tap`. Deck A and B are summed through an equal-power crossfader before master gain.

Playhead position and level meter arrive back to JS via `el.snapshot` and `el.meter` events (~30 Hz), kept in local state so they never trigger a graph re-render.

For the full design rationale, see [`SPEC.md`](SPEC.md).

---

## Scope / known gaps

- **Pitch shifts with tempo** — varispeed only; true time-stretch would require compiling `signalsmith-stretch` to WASM.
- **One stereo output** — no cue/headphone split monitoring.
- **No persistence** — in-memory session only; no IndexedDB.
- **AIFF** — works in Safari; Chrome/Firefox `decodeAudioData` support varies.
