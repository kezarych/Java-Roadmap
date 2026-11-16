# Phase 3: Courtyard with House and Annexes — Implementation Plan

> **СТАТУС 2026-11-16: ПЛАН УСТАРЕЛ.**
> Пользователь уточнил концепцию: главный экран — **интерьер дома**,
> в центре стены висит карта, клик открывает отдельный экран карты.
> Карта — **конечный путь**, не замкнутое кольцо. Всё ниже про «двор
> с кольцевой тропой» не соответствует цели. Часть наработок Task 1–3
> уже лежит в рабочем дереве (кольцо, пристройки, тропы) и подлежит
> переделке; `annexSvg`/`annexPlateSvg` можно переиспользовать.
> Новую версию плана писать после утверждения интерьера.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the old two-tab map/house split with one courtyard screen: a large house in the center, a ring trail with six stage annexes, a central fire, eight trails at the outer edge, and rooms reachable below the scene.

**Architecture:** All changes stay in `index.html` (vanilla JS + inline SVG/CSS). The ring path and annex slots are explicit constants. `annexSvg(type, state, done, total)` renders one of six buildings in one of three build states. Navigation collapses to one courtyard tab plus five room subpanels (`#panel-*`). No state-schema changes; progress comes from existing `state.tasks`.

**Tech Stack:** HTML, CSS, inline SVG, vanilla JS, Playwright for verification.

**Spec:** `docs/superpowers/specs/2026-11-15-house-redesign-design.md` (§5.1–5.3, §6.2–6.4, §7, §9, §11 phase 3, §12).

## Global Constraints

- Primary target: desktop 1280×1000. Mobile stays working (no page-level horizontal scroll) but separate `MOBILE_SLOTS` are **deferred** per user preference; do not block on mobile art.
- No new dependencies, no build step, no CDN.
- No New Year decor (ornamented trees, snow, gifts, garlands).
- State stays `version: 4`; `activity`/`frozen` retention and calendar are untouched.
- Clicking an annex opens its stage sheet; the house opens the room strip; the fire opens the calendar. No functionality is locked behind progress.
- All interactive objects: `tabindex`, Enter/Space, `aria-label`.
- Respect `prefers-reduced-motion`.
- Console must stay error-free.
- Test suites in `C:\Users\steam\AppData\Local\Temp\pw-play` must end green; update their assumptions where phase 3 intentionally changes the DOM contract.
- Commit after each task; bump `CACHE_NAME` in `sw.js` once at the end.

## Review Focus

- Fire must paint above annex plaques yet never intercept clicks meant for a plaque or annex.
- Annex + plaque + current ring + house + fire + beacon + trail labels must not intersect at 1280×1000.
- Every `.map-stage` placket opens its own stage; the house, beacon and fire each open their own target.
- Walker must sit beside the current annex without covering its number, and progress dashes must not leak past the current annex.
- Removing `#panel-house` / top tabs must not break `#practice`, `#achievements`, `#checklist`, `#settings`, `#finds` deep links or the back button.
- All eight trails must render at the outer edge and be clickable; locked ones explain the requirement instead of doing nothing.

---

### Task 1: Courtyard geometry and static scene

**Files:**
- Modify: `index.html` (constants ~5032; `MAP_TREES` ~5059; `TRAILS` ~4386; `mapSkyDecorSvg`/`mapDecorSvg` ~5701; `mapStaticSvg` ~5755; `buildMapGeo` ~5883)

**Interfaces:**
- Produces:
  - `COURT_HOUSE = { x: 500, y: 520, scale: 1.7 }`
  - `COURT_FIRE = { x: 300, y: 625, scale: 1.85 }`
  - `COURT_BEACON = { x: 900, y: 200 }`
  - `ANNEX_SLOTS = [{ id, x, y, type }]` in stage order:
    `stage1 (285,545,workshop)`, `stage2 (410,650,arena)`, `stage3 (610,655,archive)`,
    `stage4 (760,555,lab)`, `stage5 (830,420,forge)`, `stage6 (675,330,observatory)`
  - `MAP_RING_D` — closed cubic path through the six slots in order, back to the first.
  - `PLATE_W = 128`, `PLATE_H = 44`.
  - `pointForStage(index)` → `{x,y}` from `ANNEX_SLOTS`.
- Consumes: existing `MAP_W=1000`, `MAP_H=700`, `MAP_GROUND_Y=330`, `TRAILS`, `FINDS`.

- [ ] **Step 1: Write the failing layout test**

Create `C:/Users/steam/AppData/Local/Temp/pw-play/phase3.js` with Playwright helpers copied from `phase1.js` (`baseState`, `pageWithState`, `check`). Add checks:
- `ANNEX_SLOTS.length === 6`, types `["workshop","arena","archive","lab","forge","observatory"]`.
- `.map-stage` count is 6; every `data-stage-open` is unique `stage1..stage6`.
- Every `.map-plate` bbox lies within `y=200..696` and plates do not overlap each other.
- Plates do not overlap the house group, fire group, beacon group, or any `.map-annex`.
- `#mapRoad` exists and its `getTotalLength() > 1200`.
- All 8 `.map-trail` exist; labels do not overlap plates.
- At 1280×1000, `document.documentElement.scrollWidth - innerWidth <= 2`.

Run: `cd C:/Users/steam/AppData/Local/Temp/pw-play && node phase3.js`
Expected: FAIL (no `.map-annex`, old slot coordinates).

- [ ] **Step 2: Replace constants and scene geometry**

Replace `MAP_POINTS`/`MAP_D` with the Task 1 `Interfaces` constants. Use this ring (adjust control points only if the collision test demands):
`M285 545 C320 615 360 640 410 650 C485 680 545 680 610 655 C665 638 725 605 760 555 C795 505 820 462 830 420 C842 372 770 342 675 330 C590 319 505 300 430 338 C360 373 292 470 285 545 Z`

Move `MAP_TREES`, `TRAILS` anchors and `mapDecorSvg` props to the outer margin so they clear the ring, house, annexes and plaques. Keep `data-decor`/`data-prop` names (`hills`, `fence`, `barrel`, `bench`, `lamp`, `stones`, `crates`, `well`).

- [ ] **Step 3: Update static scene**

`mapStaticSvg` keeps sky, moon, stars, constellation, meteors, hills, trees, ground and the ring road (`#mapRoad`, `.map-road`, `.map-road-dark`, `.map-road-dash`). Remove the old road-scaling spans based on `MAP_POINTS`; derive light radius from `COURT_FIRE`. Keep `rect[data-ground]` y=330, height=370.

- [ ] **Step 4: Replace `buildMapGeo`**

Return `{ path, total, stageLengths: number[6] }`, where `stageLengths[i]` is the path length of the closest point on `path` to `ANNEX_SLOTS[i]` (sample the path every 4 px, take the closest). No other task computes ring lengths.

- [ ] **Step 5: Run layout test, screenshot, commit**

Run: `node phase3.js` — layout checks may still fail until Task 2 adds annexes; sky/road/trail checks must pass.
Screenshot `D:/vibe/сайт/.ui-court-base.png` at 1280×1000.
```bash
git add index.html && git commit -m "Phase 3: courtyard ring geometry and outer decor"
```

---

### Task 2: `annexSvg` — six buildings, three build states

**Files:**
- Modify: `index.html` (new helpers near `houseSvg` ~5547)

**Interfaces:**
- Produces:
  - `annexSvg(type, stateName, done, total)` → SVG group string, origin at the building's ground anchor, footprint `x ∈ [-46,46]`, `y ∈ [-78,16]`; includes a transparent `rect` hit area 92×92 and `data-state` on the outer group.
  - `annexPlateSvg(stage, done, total, slotIndex)` → SVG group string for a 128×44 plaque, positioned by `annexPlatePosition(slotIndex)`; includes `.map-plate`, `.map-title`, `.map-count`, `.map-mini-track`, `.map-mini-fill`, and a connector line to the annex.
  - `annexPlatePosition(slotIndex)` → `{x,y}` (top-left; bottom slots use plaque above, upper slots below).
  - `stagesProgress()` → `[{stage, done, total, stateName}]` where `stateName` is `empty` (0), `progress` (1..N−1), `done` (N).
- Consumes: `ROADMAP`, `state.tasks`, `taskKey`.

- [ ] **Step 1: Add three-state assertions to `phase3.js`**

Seed states with stage1 = 0 tasks, stage2 = 5/10, stage3 = 11/11. Expect:
- `[data-stage-open="stage1"] .map-annex[data-state="empty"]`, `.map-sign` exists.
- `[data-stage-open="stage2"] .map-annex[data-state="progress"]`, `.map-scaffold` exists, plate count text `5/10`.
- `[data-stage-open="stage3"] .map-annex[data-state="done"]`, no `.map-scaffold`, plate has `.is-done`.

- [ ] **Step 2: Implement `annexSvg`**

One branch per type with a distinct silhouette and material mix:
- `workshop` — low timber shed, awning, chimney smoke.
- `arena` — open pavilion, two posts, banner, sand circle.
- `archive` — stone block, shelves/scroll sign, slate roof.
- `lab` — brick lab, glass roof panel, cyan window.
- `forge` — stone forge, glowing furnace mouth, anvil.
- `observatory` — tall lighthouse/observatory, dome, light beam (this is also the finish landmark).

State overlays:
- `empty` — dashed plot, two stake posts, small sign with stage code.
- `progress` — foundation + 2 scaffold poles + partial frame + progress board.
- `done` — full building + warm window + small flag/smoke.

Colors: reuse existing palette variables/hex; no new fonts.

- [ ] **Step 3: Implement plates**

Compact 128×44 plaque, 2-line title (12 px), count (10 px), 4 px mini progress bar. Position rules:
- slots 1,2,3 (y ≥ 545): plaque above, `y = slot.y - 56`.
- slots 4,5,6 (y ≤ 555): plaque above for 4, below for 5/6 where space allows; validate with bboxes and adjust so no overlap.
- stage 6 plate may sit left of the observatory to keep the beacon label clear.

- [ ] **Step 4: Run phase3.js, screenshot, commit**

Run: `node phase3.js`; expect all annex-state checks pass.
Screenshot `D:/vibe/сайт/.ui-court-annexes.png`.
```bash
git add index.html && git commit -m "Phase 3: annex buildings with empty, scaffold and done states"
```

---

### Task 3: Compose the courtyard scene

**Files:**
- Modify: `index.html` (`mapDynamicSvg` ~5898, `renderMap` ~6083, `renderFire` ~5178)

**Interfaces:**
- Consumes: Task 1 constants, Task 2 `annexSvg`/`annexPlateSvg`/`stagesProgress`.
- Produces DOM contract per stage:
  `g.map-stage[data-stage-open="stageN"][data-state]` with
  `.map-annex`, `.map-plate`, `.map-mini-track`, `.map-mini-fill`,
  `.map-node` (at the slot anchor), `.map-node-num`, optional `.map-current-ring`.
- House group: `.map-house[data-house]` at `COURT_HOUSE` with scale.
- Fire group: `.map-campfire[data-fire]` at `COURT_FIRE`.
- Beacon: `.map-finish[data-stage-open="stage6"]` at `COURT_BEACON`.

- [ ] **Step 1: Add composition checks to `phase3.js`**

- House bbox center within 40 px of the scene center x; house bbox height ≥ 150.
- Fire center x < house bbox left + 80; fire bbox bottom ≥ 600.
- Beacon visible and `[data-stage-open="stage6"]`.
- `#mapDynamic` paint order: every `.map-stage` index < fire index < walker index.
- Fire click opens `#fireCalendar`; house click opens the room strip; stage 6 beacon click opens the stage 6 sheet.

- [ ] **Step 2: Rewrite `mapDynamicSvg`**

Build trails (all 8, `.map-trail`, `data-trail`, locked class + aria-label), beacon, six annex groups, house, fox, fire, walker placeholder. Draw order: trails → beacon → annexes → house → fox → fire (→ plates stay inside annex groups, before fire) → walker appended by `updateMapWalk`.

- [ ] **Step 3: Update `renderMap`/`renderFire`**

Keep `#mapRoad`, `#mapProgress`, `#mapWalker`, `#mapDynamic` IDs. Fire uses `COURT_FIRE` and stays after stages. House uses `COURT_HOUSE` and `houseSvg(stage)` scaled.

- [ ] **Step 4: Run phase3.js + test2.js, screenshot, commit**

Run both; fix collisions before committing.
Screenshot `D:/vibe/сайт/.ui-court-full.png`.
```bash
git add index.html && git commit -m "Phase 3: compose courtyard with central house, fire and beacon"
```

---

### Task 4: Walker and ring progress

**Files:**
- Modify: `index.html` (`updateMapWalk` ~6031, `firstIncompleteStage` ~4813)

**Interfaces:**
- Produces:
  - `currentStageIndex()` → `0..5`, or `-1` when every stage is complete.
  - `walkerTarget(lengths)` → `{ x, y }` on the ring 34 px before the current annex anchor; when `-1`, 24 px before the beacon-adjacent end.
  - `#mapProgress` `stroke-dasharray = "<len> 9999"`.
- Consumes: `buildMapGeo().stageLengths`, `#mapRoad`.

- [ ] **Step 1: Extend `phase3.js`**

For each partial progress level (0, 1, 7, 25, 40, 63 tasks):
- walker hidden only at 0 tasks; otherwise visible.
- current stage matches the first stage with incomplete tasks.
- `#mapProgress` length ≤ stage length of the current stage + 1 and > 0.
- walker does not cover `.map-node-num` of the current stage (reuse `walkcmp` geometry).
- after all 63 tasks, no `.is-current`; walker sits near the beacon.

- [ ] **Step 2: Implement `currentStageIndex` and walker placement**

Walker starts at fire and moves to each annex point along the ring, then the beacon. Animate with the existing `requestAnimationFrame` approach and `lastMapPos`; respect reduced motion (jump instead of animate).

- [ ] **Step 3: Implement progress dashes**

Dash length = ring length at the walker position. At `-1`, length = path length. `stroke-dasharray` stays `"0 9999"` at zero tasks (preserves existing test).

- [ ] **Step 4: Update `walkcmp.js`, run, commit**

Rewrite `walkcmp.js` to iterate 0..63 tasks, measure walker vs current annex `.map-node-num`, assert 0 overlaps. Run `node walkcmp.js` and `node phase3.js`.
```bash
git add index.html && git commit -m "Phase 3: walker stands at current annex with ring progress"
```

---

### Task 5: Merge tabs into the courtyard and add the finds room

**Files:**
- Modify: `index.html` (HTML topbar ~3497, bottom nav ~3808, panels ~3524/3594, boot script ~3480, `HOUSE_ROOMS` ~5520, `renderHousePanel` ~5626, `switchTab` ~6759, `bindEvents` ~7224, CSS ~3200-3350)
- Modify: `test2.js`

**Interfaces:**
- Produces:
  - `TOP_TABS` removed; `COURT_VIEW = "court"`.
  - `ROOM_TABS = ["practice", "achievements", "checklist", "settings", "finds"]`.
  - `TAB_ALIASES = { roadmap:"court", resources:"court", map:"court", house:"court", court:"court" }`.
  - `switchTab("court")` shows `#panel-roadmap`, `body.dataset.tab="court"`, `body.dataset.room="home"`.
  - `switchTab(room)` shows `#panel-<room>`; back button label `← Двор`.
  - `#courtRooms` — 5 compact room buttons (`data-house-view`) below the scene.
  - `#courtStatus` — compact line with stage name, meter, resources.
  - `#panel-finds` containing `#houseChest`.
- Consumes: `HOUSE_ROOMS`, `renderHousePanel`, `renderMap`.

- [ ] **Step 1: Update `test2.js` to the new contract**

Replace assertions that click `#topbarNav [data-top-tab]` with:
- courtyard default: `body.dataset.tab === "court"` and `#panel-roadmap` visible;
- no `[data-top-tab]` in topbar;
- `#courtRooms` has 5 buttons; clicking `[data-house-view="practice"]` opens `#panel-practice`;
- clicking back returns to courtyard;
- chest button opens `#panel-finds` and chest content is visible;
- `#house` hash opens courtyard; `#finds` opens finds.
Keep the remaining 30 checks intact (rename only where the panel moved).

- [ ] **Step 2: Remove top tabs, add court strip**

Replace topbar nav with a static courtyard label/badge (no `data-top-tab`). Replace bottom two-tab nav with a single “Двор” button plus the five-room strip under the scene. Remove `#panel-house`; move `#houseStageNum`, `#houseStageBar`, `#houseStageName`, `#houseResources` into `#courtStatus`; move `#houseChest` into `#panel-finds`; move `#houseRooms` usage to `#courtRooms`.

- [ ] **Step 3: Update navigation**

`switchTab` accepts `court`/`map`/`house`; sets `activeTab`, `activeRoom`, hides all panels, shows the target. Boot hash script and `HASH_TABS` accept `court/map/house` plus five rooms. `currentHashName("court","home") === "court"`. Deep links `#practice` etc. still work.

- [ ] **Step 4: Add `finds` to `HOUSE_ROOMS` and progress map**

Add room `{view:"finds", name:"Сундук", desc:"Находки с троп огня.", icon:<chest path>}`. Update `renderHousePanel` progress map (`finds: {done: stats.finds, total: Object.keys(FINDS).length, label: ...}`) and skip the chest render when `#houseChest` is absent.

- [ ] **Step 5: CSS**

Court room strip: grid 5 columns on desktop, horizontal scroll with 44 px targets on mobile. Keep `.house-room` styling for the strip. Remove/hide rules tied to the old top tabs and `#panel-house`; keep `.house-hero` CSS only if reused.

- [ ] **Step 6: Run `test2.js` + `phase3.js`, screenshot, commit**

Screenshot `D:/vibe/сайт/.ui-court-rooms.png`.
```bash
git add index.html test2.js && git commit -m "Phase 3: merge tabs into courtyard and add finds room"
```

---

### Task 6: Trails, labels and responsive checks

**Files:**
- Modify: `index.html` (trail rendering, `claimTrail` ~5444, CSS)
- Modify: `phase3.js`

**Interfaces:**
- Produces: all 8 trails always rendered; `.map-trail.is-locked` when `currentStreak() < TRAILS[i].days`; clicking a locked trail shows `Откроется на N-й день серии`; unlocked behaviour unchanged.
- Consumes: `TRAILS`, `state.finds`, `claimTrail`.

- [ ] **Step 1: Add trail checks**

- 8 `.map-trail` at 0 streak, 3 unlocked at streak 14; all have `aria-label`.
- Clicking a locked trail shows a toast and does not write `state.finds`.
- Trail label bboxes do not overlap `.map-plate` or `.map-annex`.
- No page-level horizontal overflow at 390×844; `.map-frame` scrolls internally.

- [ ] **Step 2: Implement locked trails and labels**

Keep the existing unlocked trail art; draw locked ones dim with a small lock mark. Update `claimTrail` to guard locked trails with the toast. Ensure keyboard activation matches click.

- [ ] **Step 3: Responsive polish**

At ≤980 px keep the 780 px scene with internal scroll; room strip scrolls horizontally; HUD stays sticky; nothing overflows the page. Do not add `MOBILE_SLOTS` (deferred).

- [ ] **Step 4: Run phase3.js, screenshot 390, commit**

```bash
git add index.html phase3.js && git commit -m "Phase 3: outer trails, locked states and mobile containment"
```

---

### Task 7: Full verification, screenshots, docs

**Files:**
- Modify: `sw.js`, `README.md`, `контекст/КОНТЕКСТ.md` (outside repo for the last one), `.ui-*.png` screenshots
- Modify: `C:/Users/steam/AppData/Local/Temp/pw-play/phase1.js` (retire old plate assumptions; point them at annex plaques)

- [ ] **Step 1: Rewrite `phase1.js` for the new DOM**

Keep its framing/fire/decor assertions; change selectors:
`.map-stage .map-plate` → same; `.map-house` → `.map-house`; scene-content selector → `.map-annex, .map-house, .map-campfire, .map-finish`. `#mapRoad` remains. Decor “off road” now means distance from `#mapRoad` ≥ 24 user units; “avoids plates” also includes `.map-annex` and house/fire boxes.

- [ ] **Step 2: Run every suite**

```bash
cd C:/Users/steam/AppData/Local/Temp/pw-play
for t in phase1.js phase2.js phase3.js test2.js migrate-gist.js vx.js vx3.js walkcmp.js; do
  echo "== $t"; node "$t" >/dev/null 2>&1; echo "exit $?"; done
```
Expected: every exit 0, no console errors.

- [ ] **Step 3: Regenerate screenshots**

Run the screenshot script (or `shots.js`); save `.ui-court-full.png`, `.ui-court-high.png`, `.ui-court-mobile.png`, `.ui-court-finds.png`. Inspect: house centered, no plate collisions, fire brightest, all six annexes readable.

- [ ] **Step 4: Bump cache and update docs**

`sw.js`: `java-roadmap-v11` → `java-roadmap-v12`. Update `README.md` with the courtyard/nav change; update `D:/vibe/контекст/КОНТЕКСТ.md` with current commit, phase status, and remaining phase 4 items (`celebrated`, one-time animations, reward copy).

- [ ] **Step 5: Commit**

```bash
git add index.html sw.js README.md .ui-court-*.png
git commit -m "Phase 3: courtyard acceptance, screenshots and docs"
```

## Self-Review

- Spec §5.1 one courtyard screen — Task 5.
- §5.2 composition (house center, ring, fire, beacon, trails, rooms strip) — Tasks 1, 3, 5.
- §5.3 six annexes and three build states — Task 2.
- §6.3 explicit slots, paint order, walker at current annex — Tasks 1, 3, 4.
- §6.4 decor/materials and sky ≤ ~25% — Task 1 (sky band retained at y≤330 for phase-1 test continuity).
- §7 three-state annex art — Task 2.
- §9 tabs merge, `#finds`, names — Task 5.
- §10 mobile containment + keyboard + reduced motion — Tasks 4, 6.
- §11 phase 3 item 5 mobile composition — explicitly deferred; containment only.
- §12 acceptance list mapped to phase3/test2 checks.
- §13 tests — Tasks 1–7; `phase1.js`/`walkcmp.js` updated because the plate/road contract intentionally changed.
