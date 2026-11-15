# Phase 2: Fire Calendar and Wood Removal — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the campfire menu and the wood resource with a GitHub-style fire calendar where any past day can be lit or extinguished, and migrate state to version 4 without losing progress.

**Architecture:** All changes in `index.html`. Add a calendar modal rendered from state, a single `toggleFireDay(key)` mutation, and a v4 `normalizeState`. Remove the radial fire menu, `feedFire`, `boost`, `manualDays`, `manualStack`, and `resources.wood`.

**Tech Stack:** HTML, CSS, inline SVG, vanilla JS, Playwright.

**Spec:** `docs/superpowers/specs/2026-11-15-house-redesign-design.md`

## Global Constraints

- Desktop 1280×1000 primary; mobile not a goal, must not regress.
- State `version: 4`; old v2/v3 states must load without losing tasks/practice/notes/checklist.
- No new dependencies; no New Year decor.
- Existing `test2.js` must stay 35/35.
- Keep the HUD quick button behavior: disabled and “Огонь горит” when today is lit.
- `#map` and `#house` tabs unchanged in this phase.
- Commit after each task.

## Interfaces

- `toggleFireDay(key: string) -> void` — toggles `state.activity[key]`, maintains `state.frozen`/`state.manualOff`, recomputes streak/stars, saves, re-renders.
- `renderFireCalendar() -> void` — draws the 53×7 grid into `#fireCalendarGrid`.
- `openFireCalendar() -> void` / `closeFireCalendar() -> void`.
- `fireCalendarData(today = new Date()) -> Array<{ key, date, state }>` where `state` is `"past" | "today" | "future" | "pad"`.
- `manualOff: Record<string, boolean>` in state.

---

### Task 1: State v4, normalization, migration

**Files:** Modify `index.html` (`createState` ~4481, `normalizeState` ~4499–4607). Test: `phase2.js`.

- [ ] **Step 1: Write the failing migration test.** Seed:
  `{version:3, tasks:{...}, resources:{wood:9,stone:3,ember:4}, rewarded:{...}, boost:{day:...,amount:2}, manualDays:{...}, manualStack:[...], activity:{...}, stars:2, ...}`
  After load assert: `version===4`, `resources.wood===undefined`, `resources.stone===3`, `resources.ember===4`, `boost===undefined`, `manualDays===undefined`, `manualStack===undefined`, `manualOff` is an object, tasks preserved.
- [ ] **Step 2: Run and see it fail.** `node phase2.js` → version stays 3, wood present.
- [ ] **Step 3: Implement.**
  - `createState`: `version: 4`, resources `{stone:0, ember:0}`, `manualOff: {}`, drop `boost`, `manualDays`, `manualStack`.
  - `normalizeState`: resources whitelist `["stone","ember"]`; copy `manualOff` boolean keys; drop removed fields; keep `frozen`, `spentFreezes`, `stars`, `finds`, `rewarded`.
  - Ensure `loadState` does not write defaults over existing values.
- [ ] **Step 4: Run and see it pass.**
- [ ] **Step 5: Commit.** `git commit -m "Phase 2: migrate state to v4"`

---

### Task 2: Remove wood, boost and the fire menu

**Files:** Modify `index.html` (HUD 3376–3402, resources 5086–5132, renderCampHud 5198–5251, house chips 5369+, bindings 6832–6839, campfire menu 5019–5035, handlers 6761–6787). Test: `phase2.js`.

- [ ] **Step 1: Write the failing test.** Assert: no element `#hudWood`, no `#hudWoodBtn`, no text `/Дрова|Подкинуть дров/` in `#campHud`, no `.camp-menu` in the SVG, and `localStorage` has no `wood` or `boost` after any save.
- [ ] **Step 2: Run and see it fail.**
- [ ] **Step 3: Implement.**
  - Remove wood from HUD stats, house chips, note text, `grantResources` (task → `stone`; practice → `ember`; daily suggestion → both), `renderCampHud`, DOM refs.
  - Delete `feedFire()` and its binding; delete `boostAmount` flame effect (fire level = streak only).
  - Delete `fireMenuOpen`, the menu markup, `data-fire-light`/`data-fire-undo` handlers and `state.manualStack` usage.
  - Keep `lightFire()` for today; `undoFire()` becomes unused — delete it and its manual bookkeeping.
  - HUD `#hudLight`: “Зажечь огонь” / disabled “Огонь горит”.
- [ ] **Step 4: Run and see it pass.**
- [ ] **Step 5: Commit.** `git commit -m "Phase 2: remove wood resource and fire menu"`

---

### Task 3: Calendar UI and day toggling

**Files:** Modify `index.html` (add modal markup near `#stageSheet` ~3652, CSS near `.stage-sheet` ~2080, JS near `renderStreak`/`renderFire`). Test: `phase2.js`.

- [ ] **Step 1: Write the failing test.**
  - Click `[data-fire]` → `#fireCalendar` visible, no `.camp-menu`.
  - `#fireCalendarGrid` has `371` cells.
  - Cell for yesterday is not active; click → `activity[yesterday]` set, streak increases by 1, cell active.
  - Click same cell again → key removed, `manualOff[yesterday]===true`, streak back.
  - Click a future cell → no change.
  - Escape closes; scrim closes; focus returns to the fire.
- [ ] **Step 2: Run and see it fail.**
- [ ] **Step 3: Implement.**
  - Markup: `<div id="fireCalendar" class="fire-calendar" hidden>` with scrim, dialog role, header (streak, lit days, freezes left, close), weekday labels, `#fireCalendarGrid`, legend.
  - `fireCalendarData()`: Monday-start weeks, 53 columns, grid ends with the current week; `pad` for days older than 365; `future` for days after today; `today` ring; `past` interactive.
  - Render `<button class="fire-day is-lit/is-frozen/is-today/is-pad" data-day="YYYY-MM-DD" aria-pressed>`; month labels at month changes.
  - `toggleFireDay(key)`: lit → remove `activity` and `frozen`, set `manualOff[key]=true`; unlit → set `activity[key]=1`, delete `frozen[key]`, delete `manualOff[key]`; then `syncStars(currentStreak())`, `renderStreak()`, `renderCampHud()`, `renderFire()`, `renderFireCalendar()`, `markChanged()`.
  - Open: fire click and Enter/Space → `openFireCalendar()`, scroll grid to the end, focus the dialog; close restores focus.
  - CSS: overlay z-index 85, desktop centered 860px card, 12px cells with 3px gap, horizontally scrollable grid, frozen/ring styles, reduced-motion friendly.
- [ ] **Step 4: Run and see it pass.**
- [ ] **Step 5: Commit.** `git commit -m "Phase 2: add GitHub-style fire calendar"`

---

### Task 4: Manual edits vs auto-freeze and retention

**Files:** Modify `index.html` (`autoFreezeMissedDay` ~5127, `bumpActivity` ~4758, prune in `syncStars`/activity). Test: `phase2.js`.

- [ ] **Step 1: Write the failing tests.**
  - Seed activity 2 days ago, `manualOff[yesterday]=true`, and enough `rewarded` for a freeze. Reload → `frozen[yesterday]` stays false, `spentFreezes===0`.
  - Seed activity for 200 distinct days → reload → all 200 remain.
- [ ] **Step 2: Run and see them fail.**
- [ ] **Step 3: Implement.**
  - `autoFreezeMissedDay` returns early when `state.manualOff[yesterdayKey()]`.
  - Raise pruning threshold from 180 to 400 days for `activity`, `frozen`, `manualOff`.
  - `toggleFireDay` never calls auto-freeze.
- [ ] **Step 4: Run and see them pass.**
- [ ] **Step 5: Commit.** `git commit -m "Phase 2: respect manual fire edits and keep 400 days"`

---

### Task 5: Verification and handoff

- [ ] Run `node test2.js` → 35/35, zero errors.
- [ ] Run `node phase2.js` → all checks pass.
- [ ] Run `node phase1.js` → 22/22 still pass.
- [ ] Run `node shots.js`; inspect the calendar open state at 1280×1000.
- [ ] Update `README.md` (no wood, calendar), `КОНТЕКСТ.md` status, this plan, and the design doc if behavior changed.
- [ ] Bump `CACHE_NAME` to `java-roadmap-v8`, commit.

## Review Focus

- Toggling a day that was frozen must not refund `spentFreezes` or leave a stale `frozen` flag.
- Auto-freeze must never undo a deliberate extinguish.
- Calendar dates must align by local time, not UTC, or today shifts for UTC+3.
- The 371 cells and the 365-day interactive window must not leak clicks into pad/future cells.
- Removing the fire menu must not leave dead event handlers that swallow clicks or keyboard focus.
