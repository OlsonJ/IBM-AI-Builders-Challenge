# Phase 4 Implementation Plan — Varispeed, Loops & Cue

## Overview

Phase 4 adds the "DJ feel" to the existing Phase 3 two-deck mixer. The three features are:

1. **Varispeed tempo** — a per-deck tempo knob that scales the transport increment, shifting both tempo and pitch together (no time-stretch).
2. **Manual loop** — the user sets an in-point and an out-point; when the loop is active the transport phase wraps inside that region using a floored-modulo.
3. **In-mix cue point** — a single stored position; pressing CUE during playback jumps to it; pressing CUE while stopped saves the playhead as the new cue.
4. **Waveform markers** — the Waveform canvas draws the cue marker, the loop-in marker, the loop-out marker, and shades the loop region.

The P3 codebase already scaffolded for all of this:
- `DeckState.tempo` exists and is sent to `buildDeckSignal` but is always 1.0.
- `buildDeckSignal` already computes `incPerSample = tempo / (totalFrames - 1)`.
- No new packages are required; this is pure Elementary graph + React state work.

---

## Sub-Task 1 — Varispeed Tempo (deck.ts + useDeck.ts + DeckControls.tsx)

**Intent**  
Wire the existing `tempo` field in `DeckState` so it is user-controllable. The Elementary
graph already uses it; we only need to (a) add a reducer action, (b) expose a `setTempo`
method from `useDeck`, and (c) add a tempo knob to `DeckControls`.

**Expected Outcomes**
- Dragging the tempo knob changes playback speed in real time (pitch shifts with it — varispeed by design).
- The knob displays the current rate (e.g. "−8%" / "+4%").
- Tempo survives play/pause but resets to 1.0 on a new track load (already done by the LOAD reducer case).

**Todo List**
1. In `src/deck.ts` — expand `DeckState` comment to note that `tempo` is now live (no code change needed; the field and graph wiring already exist).
2. In `src/useDeck.ts`:
   - Add `SET_TEMPO` to the `Action` union: `{ type: 'SET_TEMPO'; value: number }`.
   - Handle it in the reducer: clamp to a sensible range (e.g. 0.5..2.0) and set `state.tempo`.
   - Add `setTempo: (value: number) => void` to the `UseDeck` interface.
   - Implement it with `useCallback` calling `dispatch`.
3. In `src/components/DeckControls.tsx`:
   - Add a `Knob` for tempo: range 0.5..2.0, default 1.0, label "TEMPO", format as `±N%`.

**Relevant Context**
- [`DeckState`](src/deck.ts) — `tempo` field already present.
- [`buildDeckSignal`](src/deck.ts) — `incPerSample = s.tempo / (totalFrames - 1)` already uses it.
- [`reducer`](src/useDeck.ts:32) — add `SET_TEMPO` case here.
- [`UseDeck`](src/useDeck.ts:57) — add `setTempo` to the interface.
- [`DeckControls`](src/components/DeckControls.tsx) — add the knob here.
- [`Knob`](src/components/Knob.tsx) — existing rotary control, drag + double-click-to-reset.

**Status** — [x] done

---

## Sub-Task 2 — Loop State & Audio Graph (deck.ts + useDeck.ts)

**Intent**  
Add loop in/out points and a loop-active flag to `DeckState`, and restructure
`buildDeckSignal` so that when the loop is active the transport position wraps inside
`[loopIn, loopOut)` using a floored modulo:
```
loopedPos = loopIn + (raw - loopIn) - loopLen * floor((raw - loopIn) / loopLen)
```
This is the "floored mod" the SPEC requires (avoids `el.mod`'s sign behaviour).

Loop exit must re-base the transport: when the user turns the loop off, `useDeck` must
set `baseNorm` to the current `position` and bump `seekGen`, so playback continues from
where it is rather than where the raw accumulator was (the SPEC's "loop exit re-bases"
requirement).

**Expected Outcomes**
- While `loopActive` is true, playback loops seamlessly within `[loopIn, loopOut)`.
- Toggling the loop off continues playback from the current position, no jump.
- Loop in/out default to 0 and 1 (full track) when a track loads.
- All loop fields are reset on a new track load.

**Todo List**
1. In `src/deck.ts`:
   - Add three fields to `DeckState`: `loopIn: number` (0..1), `loopOut: number` (0..1), `loopActive: boolean`.
   - Update `initialDeckState` to set `loopIn: 0`, `loopOut: 1`, `loopActive: false`.
   - In `buildDeckSignal`, after computing the raw `position`:
     - If `loopActive`, wrap `position` using the floored-mod formula above, built with `el.sub`, `el.mul`, `el.floor`, and `el.div` (or equivalent Elementary primitives).
     - Pass the (possibly wrapped) position into the table reads and the snapshot tap.

2. In `src/useDeck.ts`:
   - Add actions: `SET_LOOP_IN`, `SET_LOOP_OUT`, `TOGGLE_LOOP`, `SET_LOOP_ACTIVE`.
   - `SET_LOOP_IN` / `SET_LOOP_OUT` — clamp 0..1, enforce `loopIn < loopOut`.
   - `TOGGLE_LOOP` — flips `loopActive`.
   - `SET_LOOP_ACTIVE { value: boolean }` — sets `loopActive` directly; when setting to `false`, the reducer also captures `baseNorm = current position` and bumps `seekGen` (loop exit re-base). The current position is not inside the reducer; handle this by passing `currentPosition` in the action payload.
   - Add methods to `UseDeck`: `setLoopIn`, `setLoopOut`, `toggleLoop`.
   - `toggleLoop` calls `dispatch({ type: 'SET_LOOP_ACTIVE', value: !state.loopActive, currentPosition: position })`.

**Relevant Context**
- [`buildDeckSignal`](src/deck.ts:118) — position computation is at line 128; loop wrapping inserts after it.
- [`el.accum`](src/deck.ts:128) — raw position is the result of `el.add(base, el.accum(...))`.
- SPEC §6 — floored-mod formula and loop-exit re-base explanation.
- [`reducer`](src/useDeck.ts:32) — add new cases.

**Status** — [ ] pending

---

## Sub-Task 3 — Cue Point (deck.ts + useDeck.ts)

**Intent**  
Add a single `cuePoint: number` (normalized 0..1) to `DeckState`. The cue workflow is:
- While **stopped**: pressing CUE saves the current playhead as the cue point.
- While **playing**: pressing CUE jumps to the cue point (seek + bump seekGen).
The cue point is purely a transport concept; no audio graph change is needed.

**Expected Outcomes**
- A cue point can be set at any time a track is loaded.
- CUE during playback jumps to the stored position instantly.
- Cue point resets to 0 on track load.

**Todo List**
1. In `src/deck.ts`:
   - Add `cuePoint: number` to `DeckState`, default `0` in `initialDeckState`.
2. In `src/useDeck.ts`:
   - Add actions: `SET_CUE { norm: number }` and `JUMP_TO_CUE`.
   - `SET_CUE` — sets `cuePoint: a.norm`.
   - `JUMP_TO_CUE` — sets `baseNorm: state.cuePoint`, bumps `seekGen`.
   - Add `setCue: (norm: number) => void` and `jumpToCue: () => void` to `UseDeck`.
   - `setCue` dispatches `SET_CUE` with the current `position` (passed from the call site, so `useDeck.setCue` calls `dispatch({ type: 'SET_CUE', norm: position })`).
   - `jumpToCue` dispatches `JUMP_TO_CUE` and also calls `setPosition(state.cuePoint)`.

**Relevant Context**
- [`DeckState`](src/deck.ts:27) — add `cuePoint` field.
- [`initialDeckState`](src/deck.ts:41) — set default.
- [`reducer`](src/useDeck.ts:32) — add `SET_CUE`, `JUMP_TO_CUE` cases.
- [`LOAD`](src/useDeck.ts:36) — reset `cuePoint: 0` here too.

**Status** — [ ] pending

---

## Sub-Task 4 — Transport Controls UI (DeckPanel.tsx)

**Intent**  
Add the CUE button, LOOP IN button, LOOP OUT button, and LOOP toggle to `DeckPanel`'s
transport row. These call the new methods exposed by `UseDeck`.

**Expected Outcomes**
- CUE button visible next to Play/Pause; active state shows when at cue point.
- LOOP IN / LOOP OUT buttons set the respective markers.
- LOOP button has an active/inactive visual state matching `loopActive`.

**Todo List**
1. In `src/components/DeckPanel.tsx`:
   - Import the new `UseDeck` methods: `setLoopIn`, `setLoopOut`, `toggleLoop`, `setCue`, `jumpToCue`.
   - In the transport row, add:
     - `<button>CUE</button>` — on click: if playing, `jumpToCue()`; if stopped, `setCue(position)`. Apply an `active` CSS class when `position` is within a small epsilon of `state.cuePoint`.
     - `<button>IN</button>` — on click: `setLoopIn(position)`.
     - `<button>OUT</button>` — on click: `setLoopOut(position)`.
     - `<button>LOOP</button>` — on click: `toggleLoop()`. Apply `active` CSS class when `state.loopActive`.

**Relevant Context**
- [`DeckPanel`](src/components/DeckPanel.tsx:21) — transport row at line 66.
- [`UseDeck`](src/useDeck.ts:57) — interface to be updated in Sub-Task 2 & 3.

**Status** — [ ] pending

---

## Sub-Task 5 — Waveform Markers (Waveform.tsx)

**Intent**  
Pass loop and cue marker data into the `Waveform` component and draw them on the canvas
during the per-frame blit (not during the offscreen peak render). Markers must be drawn
in the visible window's coordinate system, so only markers within the current view are
visible.

**Expected Outcomes**
- Cue point: a vertical line in a distinct colour (e.g. yellow) at the cue position.
- Loop in/out: vertical lines in a loop colour (e.g. green).
- Loop active region: a semi-transparent fill between loopIn and loopOut.
- Markers that are outside the visible window are not drawn.
- Drawing markers does not affect the offscreen peak cache (no rebuild triggered).

**Todo List**
1. In `src/components/Waveform.tsx`:
   - Extend `Props` with:
     ```ts
     cuePoint?: number;
     loopIn?: number;
     loopOut?: number;
     loopActive?: boolean;
     ```
   - In the `draw` callback (the per-frame path), after blitting the waveform and drawing
     the playhead:
     - Helper: convert a normalized track position to a canvas X coordinate using the
       same `start` / `win` window the playhead uses:
       `xFor(pos) = ((pos * total - start) / win) * cssW`
     - Draw the loop region fill (if `loopActive`): a semi-transparent rect from
       `xFor(loopIn)` to `xFor(loopOut)`.
     - Draw loop in/out lines (always, if set).
     - Draw cue line.
   - All new props are optional with sensible defaults (undefined / inactive), so Phase 3
     callers need no changes; only `DeckPanel` in Phase 4 passes them.
2. In `src/components/DeckPanel.tsx`:
   - Pass `cuePoint`, `loopIn`, `loopOut`, `loopActive` from `deck.state` into `<Waveform>`.

**Relevant Context**
- [`Waveform draw callback`](src/components/Waveform.tsx:82) — markers slot in after the
  playhead line (line 119).
- `windowFor` helper at line 72 — same coordinate transform used for markers.
- [`DeckPanel`](src/components/DeckPanel.tsx:63) — the `<Waveform>` call to be extended.

**Status** — [ ] pending

---

## Sub-Task 6 — CSS / Visual Polish (index.css)

**Intent**  
Add minimal styles for the new buttons (CUE, IN, OUT, LOOP) and their active states.
No layout overhaul — just slot them into the existing transport row and add colour cues.

**Expected Outcomes**
- All new buttons are visually consistent with existing transport buttons.
- Active states (loop on, at-cue) are clearly visible without being distracting.

**Todo List**
1. In `src/index.css`:
   - Add `.btn.active` style (e.g. highlighted border or background) for the LOOP and CUE active states.
   - Optionally add `.btn.cue` and `.btn.loop` colour variants to distinguish them from the play/pause buttons.
   - Check the transport row layout still works with the additional buttons (may need minor flex adjustments).

**Relevant Context**
- [`index.css`](src/index.css) — existing button and transport styles.

**Status** — [ ] pending

---

## Implementation Order

Sub-tasks are ordered by dependency:

```
Sub-Task 1 (Tempo)  ←─ independent; no dependencies on 2–5
Sub-Task 2 (Loop audio graph)  ←─ requires DeckState changes
Sub-Task 3 (Cue state)  ←─ requires DeckState changes (can run alongside 2)
Sub-Task 4 (Transport UI)  ←─ requires 2 and 3 complete
Sub-Task 5 (Waveform markers)  ←─ requires 2, 3, and 4 complete
Sub-Task 6 (CSS)  ←─ can run alongside 4 and 5
```

After all sub-tasks: run `npm run build` to typecheck; fix any TypeScript errors.
