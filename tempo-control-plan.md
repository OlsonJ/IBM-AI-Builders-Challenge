# Tempo Control Implementation Plan

## Overview

Implement **Sub-Task 1 (Varispeed Tempo)** from `phase4-plan.md` — the simplest, fully independent feature in Phase 4. The existing `DeckState.tempo` field is already wired into the Elementary audio graph (`incPerSample = s.tempo / (totalFrames - 1)` in `buildDeckSignal`), but there is no reducer action, no `useDeck` method, and no UI control to change it. This plan wires those three missing pieces together.

Also included: a **Git author configuration** sub-task to ensure commits made in this session are attributed to GitHub user `olsonj` so they appear in the contribution graph.

**Scope boundary:** Only Sub-Task 1 of phase4-plan.md is in scope here. Loop and Cue features are *not* part of this plan.

---

## Sub-Task A — Git Author Setup

**Intent**
The repo's `.git/config` has no `[user]` section, so commits are made under the system identity (not `olsonj`). Configuring the author at the repository level ensures every commit in this repo appears in the `olsonj` GitHub contribution graph.

**Expected Outcomes**
- `.git/config` contains a `[user]` section with `name = olsonj` and the email that `olsonj` uses on GitHub.
- Subsequent commits are attributed to that identity.

**Todo List**
1. Run `git config user.name "olsonj"` scoped to the repo.
2. Run `git config user.email "NguyenT.Joshua@gmail.com"` scoped to the repo.
4. Verify with `git config --list --local` that both fields are set.

**Relevant Context**
- `.git/config` — no `[user]` block present; remote `origin` points to `github.com/OlsonJ/IBM-AI-Builders-Challenge.git`.

**Status** — [ ] pending

---

## Sub-Task B — Reducer Action: SET_TEMPO

**Intent**
Add a `SET_TEMPO` action to `useDeck.ts` so the tempo knob has something to dispatch to. The reducer must clamp the incoming value to `[0.5, 2.0]` before storing it.

**Expected Outcomes**
- The `Action` union in `useDeck.ts` includes `{ type: 'SET_TEMPO'; value: number }`.
- The reducer `case 'SET_TEMPO'` returns the new state with `tempo` clamped to `[0.5, 2.0]`.
- The `UseDeck` interface exposes `setTempo: (value: number) => void`.
- `setTempo` is implemented with `useCallback` and returned from the hook.

**Todo List**
1. In [`src/useDeck.ts`](src/useDeck.ts) — add `{ type: 'SET_TEMPO'; value: number }` to the `Action` union type.
2. In the `reducer` function — add `case 'SET_TEMPO': return { ...s, tempo: Math.max(0.5, Math.min(2.0, a.value)) };`.
3. In the `UseDeck` interface — add `setTempo: (value: number) => void`.
4. In the `useDeck` function body — implement `const setTempo = useCallback((value: number) => dispatch({ type: 'SET_TEMPO', value }), []);`.
5. Include `setTempo` in the hook's return object.

**Relevant Context**
- [`src/useDeck.ts`](src/useDeck.ts) — `Action` union, `reducer`, `UseDeck` interface, and return object are all in this file.
- [`src/deck.ts`](src/deck.ts) — `DeckState.tempo` field already exists; `buildDeckSignal` already uses it. **No changes needed to this file.**

**Status** — [ ] pending

---

## Sub-Task C — Tempo Knob UI

**Intent**
Add a `Knob` component for tempo to `DeckControls.tsx`. The knob must display the current tempo as a percentage offset from 1× (e.g. `+8%`, `-4%`, `0%`), spanning the range `0.5×` to `2.0×`.

**Expected Outcomes**
- A "TEMPO" knob appears in the `DeckControls` mixer strip alongside the existing EQ and FILTER knobs.
- Dragging the knob changes playback speed and pitch in real time (varispeed — no time-stretch).
- The knob resets to `1.0` on double-click (handled by `Knob`'s existing `defaultValue` prop).
- The display reads `0%` at 1×, `+50%` at 1.5×, `-25%` at 0.75×, etc.
- Loading a new track resets tempo to `1.0` (already handled by the existing `LOAD` reducer case — no new code needed).

**Todo List**
1. In [`src/components/DeckControls.tsx`](src/components/DeckControls.tsx):
   - Add a `<Knob>` with `label="TEMPO"`, `min={0.5}`, `max={2.0}`, `defaultValue={1.0}`, `value={state.tempo}`, `onChange={deck.setTempo}`.
   - Add a `format` prop that converts the value to `±N%`: `(v) => { const pct = Math.round((v - 1) * 100); return pct === 0 ? '0%' : \`${pct > 0 ? '+' : ''}${pct}%\`; }`.
   - Place the knob logically alongside the other knobs (after the FILTER knob is a natural position).

**Relevant Context**
- [`src/components/DeckControls.tsx`](src/components/DeckControls.tsx) — existing knob layout (EQ + FILTER).
- [`src/components/Knob.tsx`](src/components/Knob.tsx) — accepts `label`, `value`, `min`, `max`, `defaultValue`, `onChange`, `format` props.
- [`src/deck.ts`](src/deck.ts) — `DeckState.tempo` is the value to bind.

**Status** — [ ] pending

---

## Sub-Task D — Commit & Push

**Intent**
Commit the tempo control changes to the integration branch and push so the work appears in GitHub contributions for `olsonj`.

**Expected Outcomes**
- `npm run build` passes with no TypeScript errors.
- A clean, descriptive commit is made under the `olsonj` author identity.
- The commit is pushed to `origin/integration`.

**Todo List**
1. Run `npm run build` to typecheck.
2. Fix any TypeScript errors.
3. Run `git add src/useDeck.ts src/components/DeckControls.tsx`.
4. Run `git commit -m "feat: add varispeed tempo control knob (phase 4, sub-task 1)"`.
5. Run `git push origin integration`.

**Relevant Context**
- Sub-Task A must be complete first (author identity configured).
- Sub-Tasks B and C must be complete first (code to commit).

**Status** — [ ] pending

---

## Implementation Order

```
Sub-Task A (Git author)    ←  must run first; no code deps
Sub-Task B (Reducer)       ←  independent; can run after A
Sub-Task C (Knob UI)       ←  requires B complete (needs setTempo on UseDeck)
Sub-Task D (Commit & Push) ←  requires A, B, C complete
```
