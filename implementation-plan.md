# DeckFlow Web — Phase 4 Implementation Plan
## Varispeed Tempo · Manual Loops · Cue Point · Waveform Markers

---

## High-Level Overview

Phase 4 builds on the completed Phase 3 two-deck mixer by adding the three features that
give a DJ mixer its "feel": **speed control**, **loop playback**, and **cue points**.
It also adds **visual markers** to the waveform so the DJ can see exactly where loops and
cues are set.

### What Phase 3 already has

| Component | File | Role |
|-----------|------|------|
| `DeckState` | `src/deck.ts` | All deck transport + mixer state |
| `buildDeckSignal()` | `src/deck.ts` | Builds the Elementary audio graph per deck |
| `useDeck()` | `src/useDeck.ts` | React hook: reducer state + live playhead/meter |
| `DeckPanel` | `src/components/DeckPanel.tsx` | Per-deck UI: load, waveform, play/pause, time |
| `DeckControls` | `src/components/DeckControls.tsx` | EQ knobs, filter knob, volume fader |
| `Waveform` | `src/components/Waveform.tsx` | Canvas waveform, playhead, click-to-seek, zoom |
| `App` | `src/App.tsx` | Combines both decks + crossfader + master volume |

### What Phase 4 adds

| Feature | Files touched | Complexity |
|---------|--------------|------------|
| **Varispeed Tempo** — per-deck speed knob (0.5×–2.0×) | `useDeck.ts`, `DeckControls.tsx` | Low — field already exists in state and audio graph |
| **Manual Loop** — set in/out points, toggle active, floored-mod wrap | `deck.ts`, `useDeck.ts`, `DeckPanel.tsx` | Medium — graph restructure + loop-exit re-base |
| **Cue Point** — save position, jump-to-cue | `deck.ts`, `useDeck.ts`, `DeckPanel.tsx` | Low — pure state, no graph change |
| **Waveform Markers** — cue line, loop lines, loop fill | `Waveform.tsx`, `DeckPanel.tsx` | Low — canvas draw additions |
| **CSS polish** — active states for new buttons | `src/index.css` | Low |

### Key constraints carried from Phase 3

- **No new packages.** All work is Elementary DSP graph construction + React state.
- **One render effect owns the whole graph** (`App.tsx`). Any `DeckState` change triggers
  a re-render and Elementary diffs the tree — only moved `el.const` values are patched,
  stateful nodes (filters, accum) keep their internal state.
- **High-frequency state stays out of the reducer.** `position` and `level` (~30 Hz)
  live in `useState` refs, not in the reducer, to avoid graph re-renders at audio rate.
- **Elementary graph keys.** Every `el.const`, `el.table`, `el.metro`, etc. carries a
  stable string `key` so the differ can match nodes across renders.

### Dependency order

```
Feature 1 (Tempo)       — independent; no dependencies
Feature 2 (Loop audio)  — needs DeckState extended
Feature 3 (Cue state)   — needs DeckState extended (parallel with Feature 2)
Feature 4 (UI buttons)  — needs Features 2 and 3 complete
Feature 5 (Waveform)    — needs Features 2, 3, and 4 complete
Feature 6 (CSS)         — can run alongside Features 4 and 5
```

---

## Feature 1 — Varispeed Tempo

### Intent

`DeckState.tempo` already exists and is already wired into the audio graph:

```ts
// src/deck.ts — line 122
const incPerSample = s.tempo / Math.max(1, totalFrames - 1);
```

But `tempo` is always `1.0` because there is no UI control and no reducer action for it.
This feature completes the wiring: add a reducer action, expose a hook method, and add a
knob to the mixer strip.

Varispeed means tempo and pitch move together — this is by design (SPEC §2) and avoids
the WASM time-stretch rabbit hole.

### How it works in the audio graph

The transport increment per audio sample is:

```
incPerSample = tempo / (totalFrames - 1)
```

At `tempo = 1.0` the track plays at normal speed. At `tempo = 1.08` it plays 8% faster
(and 8% higher in pitch). At `tempo = 0.92` it plays 8% slower. Elementary sees this as
a changed const value on the `${id}_inc` node and updates it without rebuilding the graph.

### Files and changes

#### `src/useDeck.ts`

1. Extend the `Action` union with:
   ```ts
   | { type: 'SET_TEMPO'; value: number }
   ```

2. Add a case in `reducer`:
   ```ts
   case 'SET_TEMPO':
     return { ...s, tempo: Math.max(0.5, Math.min(2.0, a.value)) };
   ```
   Clamped to **0.5–2.0** (half-speed to double-speed; sensible for a teaching tool).

3. Add `setTempo: (value: number) => void` to the `UseDeck` interface.

4. Implement it:
   ```ts
   const setTempo = useCallback((value: number) => dispatch({ type: 'SET_TEMPO', value }), []);
   ```

5. Return it in the hook's return object.

#### `src/components/DeckControls.tsx`

Add a `Knob` component below the FILTER knob:

```tsx
<Knob
  label="TEMPO"
  value={state.tempo}
  min={0.5}
  max={2.0}
  defaultValue={1.0}
  onChange={deck.setTempo}
  format={(v) => {
    const pct = Math.round((v - 1) * 100);
    return pct === 0 ? '0%' : `${pct > 0 ? '+' : ''}${pct}%`;
  }}
/>
```

The `Knob` component supports `defaultValue` for double-click-to-reset, so double-clicking
the tempo knob snaps back to 1.0 (normal speed).

### Expected outcomes

- Dragging TEMPO changes playback speed in real time with no audio click.
- Display reads `+8%`, `-5%`, `0%`, etc.
- Double-click resets to 0% (normal speed).
- A new track load resets tempo to 1.0 (already handled by the existing `LOAD` reducer case: `tempo: 1`).

---

## Feature 2 — Manual Loop

### Intent

Add a loop region `[loopIn, loopOut)` to `DeckState`. When the loop is active, the
transport's read position is wrapped inside this region using a **floored modulo** so
playback cycles seamlessly. When the loop is deactivated, the transport re-bases so
playback continues from the current position rather than from the raw accumulator value.

### How the floored modulo works

Elementary's `el.mod` is a C-style `fmod`, which keeps the dividend's sign. For a
phase that occasionally drifts slightly below `loopIn`, this can produce a negative
result. The floored version is sign-safe:

```
offset  = raw − loopIn
loopLen = loopOut − loopIn
loopedPos = loopIn + offset − loopLen × floor(offset / loopLen)
```

In Elementary nodes this is:
```ts
const offset   = el.sub(rawPos, el.const({ key: `${id}_lIn`,  value: loopIn  }));
const loopLen  = el.const({ key: `${id}_lLen`, value: loopOut - loopIn });
const wrapped  = el.sub(offset, el.mul(loopLen, el.floor(el.div(offset, loopLen))));
const position = el.add(el.const({ key: `${id}_lIn`, value: loopIn }), wrapped);
```

The structural shape of the graph changes when loop is toggled (the `if (s.loopActive)`
branch picks a different expression tree). Elementary handles this correctly — when the
graph structure changes, stateful nodes like `el.accum` keep their internal state
as long as their `key` is stable.

### Loop-exit re-base

When the loop is on, the raw accumulator is running ahead past `loopOut`. When the user
turns the loop off, if we just stop wrapping, playback would jump to wherever the raw
accumulator is, not where the waveform appears to be. The fix (SPEC §6):

> Loop exit re-bases the transport in JS so playback continues from the current spot.

In practice: when `toggleLoop` fires and `loopActive` is `true → false`, dispatch an
action that sets `baseNorm = position` (the live playhead, passed as part of the action
payload) and bumps `seekGen`. This resets the accumulator to zero and re-anchors `base`
to the visible position.

### Files and changes

#### `src/deck.ts`

1. Add to `DeckState`:
   ```ts
   loopIn:     number;   // normalized 0..1, default 0
   loopOut:    number;   // normalized 0..1, default 1
   loopActive: boolean;  // default false
   ```

2. Update `initialDeckState` with `loopIn: 0, loopOut: 1, loopActive: false`.

3. In `buildDeckSignal`, after computing `rawPos`:
   ```ts
   const rawPos = el.add(base, el.accum(inc, seekTrig));
   let position: NodeRepr_t;
   if (s.loopActive) {
     const lIn  = el.const({ key: `${s.id}_lIn`,  value: s.loopIn });
     const lLen = el.const({ key: `${s.id}_lLen`, value: s.loopOut - s.loopIn });
     const off  = el.sub(rawPos, lIn);
     const wrapped = el.sub(off, el.mul(lLen, el.floor(el.div(off, lLen))));
     position = el.add(lIn, wrapped);
   } else {
     position = rawPos;
   }
   ```
   Pass `position` (not `rawPos`) into the `el.table` reads and the `el.snapshot` tap.

#### `src/useDeck.ts`

1. Extend `Action` union:
   ```ts
   | { type: 'SET_LOOP_IN';  norm: number }
   | { type: 'SET_LOOP_OUT'; norm: number }
   | { type: 'SET_LOOP_ACTIVE'; value: boolean; currentPosition: number }
   ```

2. Add reducer cases:
   ```ts
   case 'SET_LOOP_IN':
     return { ...s, loopIn: Math.max(0, Math.min(s.loopOut - 0.001, a.norm)) };
   case 'SET_LOOP_OUT':
     return { ...s, loopOut: Math.max(s.loopIn + 0.001, Math.min(1, a.norm)) };
   case 'SET_LOOP_ACTIVE':
     if (!a.value) {
       // Loop exit: re-base transport to current visible position
       return { ...s, loopActive: false, baseNorm: a.currentPosition, seekGen: s.seekGen + 1 };
     }
     return { ...s, loopActive: true };
   ```

3. Extend `UseDeck` interface:
   ```ts
   setLoopIn:  (norm: number) => void;
   setLoopOut: (norm: number) => void;
   toggleLoop: () => void;
   ```

4. Implement with `useCallback`:
   ```ts
   const setLoopIn  = useCallback((norm: number) => dispatch({ type: 'SET_LOOP_IN',  norm }), []);
   const setLoopOut = useCallback((norm: number) => dispatch({ type: 'SET_LOOP_OUT', norm }), []);
   const toggleLoop = useCallback(() => {
     dispatch({ type: 'SET_LOOP_ACTIVE', value: !state.loopActive, currentPosition: position });
   }, [state.loopActive, position]);
   ```
   Note: `toggleLoop` captures `position` from the hook's render scope (the live playhead ref).

5. Update `LOAD` reducer case to reset loop fields:
   ```ts
   case 'LOAD':
     return { ...s, track: a.track, playing: false, baseNorm: 0, seekGen: s.seekGen + 1,
              tempo: 1, loopIn: 0, loopOut: 1, loopActive: false };
   ```

### Expected outcomes

- While `loopActive` is true, the playhead bounces seamlessly at `loopOut` back to `loopIn`.
- Toggling loop off continues from the current visible position — no jump.
- New track load clears all loop state.
- Loop in must always be less than loop out (enforced by clamp in reducer).

---

## Feature 3 — In-Mix Cue Point

### Intent

A **cue point** is a stored normalized position (0–1) in the track. The workflow mirrors
standard DJ hardware:

- **While stopped** → pressing CUE saves the current playhead as the cue point.
- **While playing** → pressing CUE instantly seeks to the cue point.

No audio graph change is required — cue is entirely a transport concept handled in the
reducer and UI.

### Files and changes

#### `src/deck.ts`

1. Add to `DeckState`:
   ```ts
   cuePoint: number;   // normalized 0..1, default 0
   ```

2. Update `initialDeckState`: `cuePoint: 0`.

#### `src/useDeck.ts`

1. Extend `Action` union:
   ```ts
   | { type: 'SET_CUE';    norm: number }
   | { type: 'JUMP_TO_CUE' }
   ```

2. Add reducer cases:
   ```ts
   case 'SET_CUE':
     return { ...s, cuePoint: Math.max(0, Math.min(1, a.norm)) };
   case 'JUMP_TO_CUE':
     return { ...s, baseNorm: s.cuePoint, seekGen: s.seekGen + 1 };
   ```

3. Extend `UseDeck` interface:
   ```ts
   setCue:    (norm: number) => void;
   jumpToCue: () => void;
   ```

4. Implement:
   ```ts
   const setCue    = useCallback((norm: number) => dispatch({ type: 'SET_CUE', norm }), []);
   const jumpToCue = useCallback(() => {
     setPosition(state.cuePoint);
     dispatch({ type: 'JUMP_TO_CUE' });
   }, [state.cuePoint]);
   ```

5. Update `LOAD` reducer case to reset cue:
   ```ts
   case 'LOAD':
     return { ...s, track: a.track, ..., cuePoint: 0 };
   ```

### Expected outcomes

- When stopped: pressing CUE pins the current playhead as the saved cue position.
- When playing: pressing CUE jumps to the saved cue position instantly.
- Cue resets to 0 on track load.

---

## Feature 4 — Transport Controls UI

### Intent

Wire the new `UseDeck` methods to UI buttons in `DeckPanel`. The transport row gains
four new buttons: **CUE**, **IN**, **OUT**, and **LOOP**.

### Button behaviour

| Button | While stopped | While playing | Active state |
|--------|--------------|---------------|-------------|
| **CUE** | Saves playhead as cue point | Jumps to cue point | When `position ≈ cuePoint` (within ε = 0.001) |
| **IN** | Sets `loopIn` to current position | Sets `loopIn` to current position | Never |
| **OUT** | Sets `loopOut` to current position | Sets `loopOut` to current position | Never |
| **LOOP** | Activates/deactivates loop | Activates/deactivates loop | When `state.loopActive` is `true` |

### Files and changes

#### `src/components/DeckPanel.tsx`

The transport `<div>` currently contains only the play/pause button and the time display.
Extend it:

```tsx
<div className="transport">
  <button
    className={`btn ghost ${Math.abs(deck.position - deck.state.cuePoint) < 0.001 ? 'active' : ''}`}
    onClick={() => deck.state.playing ? deck.jumpToCue() : deck.setCue(deck.position)}
    disabled={!track}
  >
    CUE
  </button>

  <button className="btn ghost" onClick={() => deck.setLoopIn(deck.position)} disabled={!track}>
    IN
  </button>
  <button className="btn ghost" onClick={() => deck.setLoopOut(deck.position)} disabled={!track}>
    OUT
  </button>
  <button
    className={`btn ghost ${deck.state.loopActive ? 'active' : ''}`}
    onClick={deck.toggleLoop}
    disabled={!track}
  >
    LOOP
  </button>

  <button
    className={`btn ${deck.state.playing ? 'stop' : 'start'}`}
    onClick={deck.togglePlay}
    disabled={!track}
  >
    {deck.state.playing ? '◼ Pause' : '▶ Play'}
  </button>

  <span className="time">
    {track ? `${fmt(deck.position * track.duration)} / ${fmt(track.duration)}` : '0:00 / 0:00'}
  </span>
</div>
```

### Expected outcomes

- All four new buttons appear in the transport row for each deck.
- LOOP button shows a highlighted state when active.
- CUE button shows a highlighted state when the playhead is at the cue point.
- All buttons disabled when no track is loaded.

---

## Feature 5 — Waveform Markers

### Intent

Draw cue and loop markers directly on the `Waveform` canvas during the per-frame blit.
Markers must respect the current zoom window so they stay aligned with the waveform
content. The offscreen peak cache must **not** be rebuilt — markers are drawn on the live
canvas after blitting the cached bitmap.

### Colour scheme (matching existing waveform palette)

| Marker | Colour | Style |
|--------|--------|-------|
| Playhead | `#ff6b6b` (existing) | 2px vertical line |
| Cue point | `#ffcc33` (yellow) | 2px vertical line |
| Loop in  | `#4cde80` (green) | 2px vertical line |
| Loop out | `#4cde80` (green) | 2px vertical line |
| Loop region | `rgba(76, 222, 128, 0.15)` | filled rect from loopIn to loopOut |

### Coordinate transform

The same transform already used for the playhead:
```ts
xFor(norm: number) = ((norm * total - start) / win) * cssW
```
where `total` = offscreen canvas pixel width (one pixel per peak bucket), `start` / `win`
come from the `windowFor()` helper, and `cssW` is the on-screen CSS width.

### Files and changes

#### `src/components/Waveform.tsx`

1. Extend `Props`:
   ```ts
   interface Props {
     peaks:      TrackPeaks | null;
     position:   number;
     onSeek:     (norm: number) => void;
     cuePoint?:  number;        // normalized 0..1
     loopIn?:    number;        // normalized 0..1
     loopOut?:   number;        // normalized 0..1
     loopActive?: boolean;
   }
   ```
   All new props are optional — Phase 3 callers need zero changes.

2. In the `draw` callback, after the existing playhead line, add:

   ```ts
   const xFor = (norm: number) => ((norm * total - start) / win) * cssW;

   // Loop region fill
   if (loopActive && loopIn !== undefined && loopOut !== undefined) {
     ctx.fillStyle = 'rgba(76, 222, 128, 0.15)';
     const x1 = Math.max(0, xFor(loopIn));
     const x2 = Math.min(cssW, xFor(loopOut));
     if (x2 > x1) ctx.fillRect(x1, 0, x2 - x1, cssH);
   }

   // Loop in/out lines
   if (loopIn !== undefined) {
     const lx = xFor(loopIn);
     if (lx >= 0 && lx <= cssW) {
       ctx.strokeStyle = '#4cde80';
       ctx.lineWidth = 2;
       ctx.beginPath(); ctx.moveTo(lx, 0); ctx.lineTo(lx, cssH); ctx.stroke();
     }
   }
   if (loopOut !== undefined) {
     const lx = xFor(loopOut);
     if (lx >= 0 && lx <= cssW) {
       ctx.strokeStyle = '#4cde80';
       ctx.lineWidth = 2;
       ctx.beginPath(); ctx.moveTo(lx, 0); ctx.lineTo(lx, cssH); ctx.stroke();
     }
   }

   // Cue line
   if (cuePoint !== undefined) {
     const cx = xFor(cuePoint);
     if (cx >= 0 && cx <= cssW) {
       ctx.strokeStyle = '#ffcc33';
       ctx.lineWidth = 2;
       ctx.beginPath(); ctx.moveTo(cx, 0); ctx.lineTo(cx, cssH); ctx.stroke();
     }
   }
   ```

3. The `draw` `useCallback` dependency array must include `cuePoint`, `loopIn`,
   `loopOut`, `loopActive` so the canvas updates when any of them change.

#### `src/components/DeckPanel.tsx`

Update the `<Waveform>` call:
```tsx
<Waveform
  peaks={track?.peaks ?? null}
  position={deck.position}
  onSeek={deck.seek}
  cuePoint={deck.state.cuePoint}
  loopIn={deck.state.loopIn}
  loopOut={deck.state.loopOut}
  loopActive={deck.state.loopActive}
/>
```

### Expected outcomes

- Yellow vertical line at the cue point, visible in the current zoom window.
- Green vertical lines at loop in and loop out.
- Semi-transparent green fill between loop in and out when loop is active.
- All markers scroll with the waveform as the user zooms and seeks.
- No impact on the cached offscreen peaks bitmap.

---

## Feature 6 — CSS / Visual Polish

### Intent

Add minimal styles for the new button states. No layout overhaul — the existing
`.transport` flex row simply needs the new buttons to look distinct from play/pause.

### Colour conventions (matching existing palette)

- `--accent`: `#4cc2ff` (cyan) — existing active/primary colour
- New: **active** state uses a warm tint or border to indicate "on"

### Changes to `src/index.css`

1. Add an `.active` modifier to `.btn`:
   ```css
   .btn.active {
     outline: 2px solid var(--accent);
     outline-offset: 2px;
   }
   ```

2. Add a `.btn.cue` variant (yellow tint for the CUE button when active):
   ```css
   .btn.cue.active {
     outline-color: #ffcc33;
   }
   ```

3. Add a `.btn.loop` variant (green tint for the LOOP button when active):
   ```css
   .btn.loop.active {
     outline-color: #4cde80;
   }
   ```

4. Check the `.transport` flex row — the four new small buttons may need `flex-wrap: wrap`
   or a reduced `gap` if the row overflows on narrower screens.

### Expected outcomes

- LOOP button glows green when active.
- CUE button glows yellow when at the cue point.
- All other buttons remain visually unchanged.

---

## Implementation Order & Validation

### Recommended order

| Step | Sub-task | Reason |
|------|----------|--------|
| 1 | Feature 1 (Tempo) | Zero dependencies; quick win |
| 2 | Feature 2 (Loop audio graph) | Core graph change; must land before UI |
| 3 | Feature 3 (Cue state) | State-only; can be done in the same commit as Feature 2 |
| 4 | Feature 4 (Transport UI) | Requires Features 2 + 3 |
| 5 | Feature 5 (Waveform markers) | Requires Features 2, 3, 4 |
| 6 | Feature 6 (CSS) | Can be done alongside Features 4 + 5 |

### Validation

After all features are implemented:

```bash
npm run build    # TypeScript strict-mode compile + Vite bundle
```

All TypeScript errors must be resolved. In particular:
- Every new `DeckState` field must be present in `initialDeckState` and in the `LOAD` reducer case.
- Every new `UseDeck` interface method must be returned by `useDeck()`.
- `Waveform`'s `draw` callback dependency array must list all props that markers read.

---

## Files Changed — Summary

| File | Changes |
|------|---------|
| `src/deck.ts` | Add `loopIn`, `loopOut`, `loopActive`, `cuePoint` to `DeckState`; update `initialDeckState`; restructure `buildDeckSignal` for loop wrapping |
| `src/useDeck.ts` | Add `SET_TEMPO`, `SET_LOOP_IN`, `SET_LOOP_OUT`, `SET_LOOP_ACTIVE`, `SET_CUE`, `JUMP_TO_CUE` actions and reducer cases; expose 6 new methods on `UseDeck` |
| `src/components/DeckControls.tsx` | Add TEMPO knob |
| `src/components/DeckPanel.tsx` | Add CUE, IN, OUT, LOOP buttons; pass marker props to `<Waveform>` |
| `src/components/Waveform.tsx` | Extend props; draw cue line, loop lines, loop fill in `draw` |
| `src/index.css` | Add `.btn.active`, `.btn.cue.active`, `.btn.loop.active` |

No new files. No new packages.
