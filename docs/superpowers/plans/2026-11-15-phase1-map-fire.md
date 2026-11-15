# Phase 1: Map Framing, Plates, Fire — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the courtyard map screen feel centered and alive: no dead sky/bottom, no overlapping stage plates, and a fire that clearly grows with the streak.

**Architecture:** All changes stay in the single-file `index.html` (vanilla JS + inline SVG/CSS). Add one small plate-layout helper, move the campfire above stage plates, enrich the static scene layers, and adjust the map frame's CSS. No state/schema changes in this phase.

**Tech Stack:** HTML, CSS, inline SVG, vanilla JS, Playwright for verification.

**Spec:** `docs/superpowers/specs/2026-11-15-house-redesign-design.md`

## Global Constraints

- Primary target: desktop 1280×1000. Mobile is **out of scope for now**; do not deliberately break it.
- No new dependencies, no build step, no CDN.
- No New Year decor (trees with ornaments, snow, gifts, garlands).
- State schema stays `version: 3` in this phase. No `wood`/calendar changes yet.
- Existing E2E suite (`test2.js`) must stay at 35/35 with zero console errors.
- Respect `prefers-reduced-motion`.
- Commit after each task.

## Review Focus

- The stage plate collision solver must also handle a stage with long wrapped text and the current-stage ring; a wrong guess clips the ring under a neighboring plate.
- Fire must remain clickable after being painted above plates; a wrong guess makes the fire swallow stage clicks or vice versa.
- The horizon/ground change must not cover or detach existing trails, fox, fireflies, house, or beacon.
- Decor props must not be placed on the road or inside plate boxes.
- Scene crop must not cut the moon, stars, stage 5 plate, or the finish beacon.

---

### Task 1: Scene balance and frame centering

**Files:**
- Modify: `index.html` (map static SVG around 5499–5520; map CSS around 2754–2763 and 3068–3076)
- Test: `C:\Users\steam\AppData\Local\Temp\pw-play\phase1.js`

**Interfaces:**
- Produces: `MAP_GROUND_Y = 330` constant used by static scene and tests.
- Produces: CSS custom property `--map-horizon` not required; keep constants in JS.

- [ ] **Step 1: Write the failing test**

In `phase1.js`, seed a full-progress state and assert:

```js
const scene = await page.evaluate(() => {
  const svg = document.getElementById('journeyMap');
  const rects = [...svg.querySelectorAll('.map-stage .map-plate, .map-house, .map-campfire')];
  const boxes = rects.map((el) => el.getBBox());
  const minY = Math.min(...boxes.map((b) => b.y));
  const maxY = Math.max(...boxes.map((b) => b.y + b.height));
  const ground = svg.querySelector('rect[data-ground]');
  return { minY, maxY, groundY: ground ? Number(ground.getAttribute('y')) : null, mapH: 700 };
});
expect(scene.groundY).toBe(330);
expect(scene.maxY).toBeGreaterThanOrEqual(650);
expect(scene.minY).toBeLessThanOrEqual(280);
const frame = await page.locator('#mapFrame').boundingBox();
expect(frame.y).toBeLessThanOrEqual(210);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node phase1.js`
Expected: FAIL — `groundY` is 398, `minY` ≈ 294, `frame.y` ≈ 242.

- [ ] **Step 3: Implement**

- Add `const MAP_GROUND_Y = 330;` near `MAP_W`/`MAP_H` (around index.html:4889).
- Replace the ground rect `y="398"` with `data-ground="1" y="${MAP_GROUND_Y}" height="${MAP_H - MAP_GROUND_Y}"`.
- Move the horizon group (`${horizon}` around 5508) and its generated hills to sit at `MAP_GROUND_Y`; keep the moon at `(858, 100)`.
- Add a soft mist band at `y = MAP_GROUND_Y - 18` and a second ground band at `MAP_GROUND_Y + 54`.
- Crop 24 px of empty sky via `renderMap`: `dom.journeyMap.setAttribute("viewBox", "0 24 " + MAP_W + " " + (MAP_H - 24));` (around 5674). Keep `MAP_H` for layout math.
- CSS (near 2756): reduce the dead top gap:

```css
#panel-roadmap { margin-top: 0; }
.map-frame { margin: 0 auto; }
@media (min-width: 981px) {
  .app-main { padding-top: 4px; }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `node phase1.js`
Expected: PASS for the balance assertions.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Phase 1: rebalance map horizon and frame"
```

---

### Task 2: Non-overlapping stage plates

**Files:**
- Modify: `index.html` (mapDynamicSvg around 5556–5591, plate CSS around 2786+)
- Test: `phase1.js`

**Interfaces:**
- Produces: `layoutStagePlates(points: Array<{x:number,y:number}>) -> Array<{x:number,y:number,w:number,h:number,above:boolean}>`
- Produces: constants `PLATE_W = 140`, `PLATE_H = 64`, `PLATE_GAP = 10`.
- Consumes: `MAP_POINTS`, `mapGeo.points`, `MAP_W`, `MAP_H`.

- [ ] **Step 1: Write the failing test**

```js
const plates = await page.evaluate(() => {
  const svg = document.getElementById('journeyMap');
  return [...svg.querySelectorAll('.map-stage')].map((g) => {
    const r = g.querySelector('.map-plate').getBBox();
    return { x: r.x, y: r.y, w: r.width, h: r.height };
  });
});
for (let i = 0; i < plates.length; i += 1) {
  for (let j = i + 1; j < plates.length; j += 1) {
    const a = plates[i], b = plates[j];
    const overlapX = Math.min(a.x + a.w, b.x + b.w) - Math.max(a.x, b.x);
    const overlapY = Math.min(a.y + a.h, b.y + b.h) - Math.max(a.y, b.y);
    expect(overlapX <= 0 || overlapY <= 0).toBe(true);
  }
}
for (const p of plates) {
  expect(p.y).toBeGreaterThanOrEqual(200);
  expect(p.y + p.h).toBeLessThanOrEqual(696);
}
```

Also assert the current-stage ring's bbox does not intersect any other plate.

- [ ] **Step 2: Run test to verify it fails**

Run: `node phase1.js`
Expected: FAIL — plates 5 and 6 vertically overlap or are 4 px apart, and the ring intersects.

- [ ] **Step 3: Implement**

- Add `layoutStagePlates(points)` near `mapDynamicSvg`:
  - assign `above = index % 2 === 0`;
  - candidate `x` clamped to `[10, MAP_W - PLATE_W - 10]`;
  - candidate `y = above ? point.y - 52 - PLATE_H : point.y + 40`;
  - iterate pairs sorted by `y`; if rectangles intersect within `PLATE_GAP`, push the later plate down (`y = other.y + other.h + PLATE_GAP`); if it exceeds `MAP_H - PLATE_H - 12`, push it above the other plate;
  - clamp `y` to `[200, MAP_H - PLATE_H - 12]`;
  - return positions.
- Use the helper in `mapDynamicSvg` instead of `px/py`.
- Compact plate content to 140×64: icon 20×20 at `(x+14, y+13)`; title 2 lines at `x+34, y+22`; count at `x+34, y+45`; mini track at `y+54`; keep the connector from node to plate edge.
- Keep `map-current-ring` at the node; if it intersects a neighboring plate, nudge the ring's neighbors via the same solver (add ring box `point ± 22` to collision checks).

- [ ] **Step 4: Run test to verify it passes**

Run: `node phase1.js`
Expected: PASS for all pairwise and ring checks.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Phase 1: add non-overlapping stage plate layout"
```

---

### Task 3: Fire above cards and visibly stronger

**Files:**
- Modify: `index.html` (`mapCampfireInner` around 4981–5028; dynamic order around 5620–5624; fire CSS around 2800–2808, 2380–2383)
- Test: `phase1.js`

**Interfaces:**
- Produces: `const CAMPFIRE_SCALE = 1.85;`
- Produces: `glowRadius(level) = 18 + level * 6`.
- Consumes: `findFireStage(streak)`, `mapCampfireInner(streak, lit)`.

- [ ] **Step 1: Write the failing test**

```js
async function fireInfo(streak) {
  return page.evaluate((s) => {
    const svg = document.getElementById('journeyMap');
    const fire = svg.querySelector('[data-fire]');
    const flames = fire.querySelectorAll('.camp-flame').length;
    const glow = fire.querySelector('.camp-glow');
    const order = [...svg.querySelectorAll('#mapDynamic > g')].map((g) => g.getAttribute('class') || g.id);
    return {
      flames,
      glowR: glow ? Number(glow.getAttribute('r')) : 0,
      fireIndex: order.findIndex((c) => String(c).includes('map-campfire')),
      lastStageIndex: Math.max(...[...svg.querySelectorAll('.map-stage')].map((g) => [...g.parentNode.children].indexOf(g)))
    };
  }, streak);
}
const low = await fireInfo(3);
const high = await fireInfo(60);
expect(high.flames).toBeGreaterThan(low.flames);
expect(high.glowR).toBeGreaterThan(low.glowR);
expect(high.fireIndex).toBeGreaterThan(high.lastStageIndex);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node phase1.js`
Expected: FAIL — fire is painted before `.map-stage` groups; flames cap at 6 but glow growth is small.

- [ ] **Step 3: Implement**

- In `mapDynamicSvg`, return `${trails}${house}${fox}${stages}<g class="map-campfire">…` so the fire paints above plates.
- Scale the campfire group with `CAMPFIRE_SCALE`; add a ground light ellipse `<ellipse class="camp-light" rx="${56 + level*8}" ry="${18 + level*2}" cy="30" fill="var(--fire)" opacity=".18"/>` before logs.
- Change glow radius to `glowRadius(level)`, opacity to `lit ? 0.24 : 0.1`.
- Make flame scale `0.48 + 0.09 * level` and allow up to 7 flames at stage 100.
- Add an always-on drop shadow to `.map-campfire` in CSS: `filter: drop-shadow(0 0 10px rgba(255,154,46,.35));` plus a stronger `.is-lighting` rule.
- Keep the phase-2 menu untouched for now; ensure it still paints above plates.

- [ ] **Step 4: Run test to verify it passes**

Run: `node phase1.js`
Expected: PASS for flames, glow, and DOM order.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Phase 1: make campfire brighter and topmost"
```

---

### Task 4: Bottom and sky decor

**Files:**
- Modify: `index.html` (static scene around 5499–5520; new `mapDecorSvg` helper near mapStaticSvg; CSS around 2770–2880)
- Test: `phase1.js`, `shots.js`

**Interfaces:**
- Produces: `mapDecorSvg() -> string` injected after the road layers and before `${stars}`/`${eyes}` as appropriate.
- Props must carry `data-decor="1"` and `data-prop="fence|barrel|bench|lamp|stones|hills"`.

- [ ] **Step 1: Write the failing test**

```js
const decor = await page.evaluate(() => {
  const svg = document.getElementById('journeyMap');
  const props = [...svg.querySelectorAll('[data-decor]')];
  return {
    count: props.length,
    types: [...new Set(props.map((p) => p.getAttribute('data-prop')))],
    below600: props.filter((p) => p.getBBox().y > 600).length,
    onRoad: props.some((p) => {
      const b = p.getBBox();
      const road = document.getElementById('mapRoad').getBBox();
      return !(b.x + b.width < road.x || b.x > road.x + road.width || b.y + b.height < road.y || b.y > road.y + road.height);
    })
  };
});
expect(decor.count).toBeGreaterThanOrEqual(8);
expect(decor.types).toEqual(expect.arrayContaining(['fence', 'barrel', 'bench', 'lamp', 'stones', 'hills']));
expect(decor.below600).toBeGreaterThanOrEqual(4);
expect(decor.onRoad).toBe(false);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `node phase1.js`
Expected: FAIL — no `[data-decor]` elements.

- [ ] **Step 3: Implement**

- Add `mapDecorSvg()` returning hand-placed pixel props:
  - distant hills at `y ≈ 290–330` (`data-prop="hills"`, no New Year);
  - fence segments and path stones below `y=600`;
  - barrel near the house, bench near the fire, lamp post near the road;
  - keep all props outside plate boxes, trail marks, and the house bbox.
- Inject into `mapStaticSvg` after the tree layers and before the fire vignette.
- Add CSS: muted wood/stone palette, subtle `filter: drop-shadow(0 1px 0 rgba(0,0,0,.4))`, no animations except the lamp flame (respect reduced motion).
- Do not place props on the road; verify by bbox test.

- [ ] **Step 4: Run test to verify it passes**

Run: `node phase1.js && node shots.js`
Expected: PASS and new `.ui-map-full.png` shows balanced top/bottom without overlap.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Phase 1: add map decor and fill bottom band"
```

---

### Task 5: Full verification

**Files:**
- Test: `test2.js`, `phase1.js`, `migrate-gist.js`, `shots.js` (temp folder)

- [ ] **Step 1: Run the existing suite**

Run: `node test2.js`
Expected: 35/35, zero console errors.

- [ ] **Step 2: Run the new phase tests**

Run: `node phase1.js`
Expected: all assertions pass at 1280×1000.

- [ ] **Step 3: Check mobile does not regress**

Run: `node test2.js` mobile cases.
Expected: layout still usable, no body horizontal overflow. Do not spend time polishing mobile in this phase.

- [ ] **Step 4: Regenerate screenshots**

Run: `node shots.js`
Expected: `.ui-map-high.png` and `.ui-map-full.png` show no plate overlap, visible fire, filled bottom band.

- [ ] **Step 5: Bump cache and commit**

```bash
git add sw.js
git commit -m "Phase 1: bump service worker cache"
```

Update `CACHE_NAME` to `java-roadmap-v7` before committing.

## Self-Review

- Spec coverage: framing (Task 1), overlaps (Task 2), fire (Task 3), decor (Task 4), verification (Task 5).
- Mobile intentionally deferred by user; only regression check remains.
- State, wood, calendar, annexes are out of scope for this plan.
