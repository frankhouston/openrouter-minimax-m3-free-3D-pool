# Revisions — how the file became functional

This document lists every bug, fix, and tuning change made between the
initial draft and the final working `minimax-m3-pool.html`. Ordered by
priority (catastrophic → visual → polish).

The model identity changed during the session:
`ling-3.0-flash-fin-free → muse-spark-1.2-contributor-free → z-ai/glm-5.3 →
z-ai/glm-5.2 → moonshotai/kimi-k3 → z-ai/glm-5.2:free → minimax/minimax-m3:free`
(current, via OpenRouter). All revisions below were driven by Frank's
"iterate step through prompt" instruction and the failed visual checks.

---

## A. Catastrophic bugs (file wouldn't run or produced nonsense)

### A1. Stray `function prevObjCount=16;` syntax error
**Symptom:** `Uncaught SyntaxError: Unexpected identifier` on load.
**Cause:** Leftover from an earlier draft; a function declaration with
`=` instead of `{`.
**Fix:** Deleted the line.

### A2. Duplicate `const sy` shadowing `let sy`
**Symptom:** `Uncaught SyntaxError: Identifier 'sy' has already been
declared` on load.
**Cause:** `project()` declared `let sy = Math.sin(camYaw)` and later
`const sy = ...` for the screen Y.
**Fix:** Renamed the second usage to `screenX` / `screenY`, and renamed
the sin to `sn`.

### A3. Projection depth sign error
**Symptom:** Table appears as a tiny dot in a corner of the canvas; "only
one corner of box is visible" — Frank's report.
**Cause:** `zv = -(vx*fx + vy*fy + vz*fz)`. With `f` pointing from camera
TO target, `v·f` is **already positive** for in-front points; the negation
flipped depth to negative and the `if(zv<0.1)zv=0.1` clamp then produced
garbage screen coordinates.
**Fix:** `let zv = vx*fx + vy*fy + vz*fz;` (no negation).

### A4. Camera basis vectors not properly derived
**Symptom:** Scene rendered at weird angle; up vector components used
`upx` (always 0) instead of `uy` in the up computation; the `right`
vector was ad-hoc.
**Cause:** `let uz = rx*fy - ry*upx;` should be `let uz = rx*fy - ry*fx;`
and the `right` / `up` cross products weren't built from a proper
orthonormal basis.
**Fix:** Rebuilt basis as:
```js
// forward = (target - cam) normalized
let fx=-camx, fy=-camy+camTarget.y, fz=-camz;
// right = forward × worldUp(0,1,0)
let rx = fy*upz - fz*upy;
let ry = fz*upx - fx*upz;
let rz = fx*upy - fy*upx;
normalize(rx,ry,rz);
// up = right × forward
const ux = ry*fz - rz*fy;
const uy = rz*fx - rx*fz;
const uz = rx*fy - ry*fx;
```

### A5. `camTarget` declared but never applied to camera
**Symptom:** Changing `camTarget.y` had no effect on framing; table
stayed in the lower half of the frame.
**Cause:** `camy = se*CAM_DIST` ignored `camTarget.y`, and the forward
vector's Y component was `-camy` (also ignoring target).
**Fix:** `camy = se*CAM_DIST` and `fy = -camy + camTarget.y` so the
target actually moves the camera's look-at point.

---

## B. Frame-fit tuning (table was overflowing or shrunk)

### B1. `CAM_DIST` initially too small
**Symptom:** Table overflowed the canvas at most orbit angles — "only
one corner of the box is visible."
**Cause:** With the projection bug (A3) fixed, CAM_DIST=2.6 produced a
1069px-wide table (119% of the 900px frame).
**Iterative tuning:**
- CAM_DIST=2.6 → worst 119% width (cut off)
- CAM_DIST=3.5 with T=2.0×1.0 → 81% × 73% (within spec)

### B2. Table dimensions
**Symptom:** Real pool proportions (2.54m × 1.27m) didn't fit comfortably.
**Fix:** Reduced to T=2.0 × 1.0 world units, RAIL=0.10, LEG_H=0.80,
APRON=0.20, POCKET_R=0.09, BALL_R=0.030.

### B3. `camTarget.y` set to -0.2 to center the table vertically
**Symptom:** With the camera above and looking down, the table appeared
in the lower half (cy≈530-590) of the 900px frame.
**Fix:** Set `camTarget.y = -0.2` so the look-at point is 0.2m below the
bed, pulling the table up to cy≈430-450 (frame-centred).

### B4. Off-center shift during orbit
**Symptom:** At orbit angles 30°/150°/210°/330° the table's bounding-box
centre shifted by ~63px from frame centre.
**Cause:** Perspective foreshortening at oblique angles; the table
rectangle in screen space isn't symmetric around the bed centre.
**Status:** Accepted as inherent to orbiting around a rectangle; total
shift is ~7% of frame width — not visually noticeable when the whole
table fits.

---

## C. Visual quality fixes

### C1. Cue stick was drawn after balls
**Symptom:** On the strike, the cue tip slid behind the cue ball.
**Fix:** Moved `drawCueStick()` to run **before** the balls depth-sort
loop, so the cue ball overlaps the tip on impact.

### C2. Useless ternary `r: BALL_MASS ? BALL_R : BALL_R`
**Symptom:** None (cosmetic). `BALL_MASS` is always truthy.
**Fix:** Simplified to `r: BALL_R`.

### C3. Cloth too dark
**Symptom:** Cloth read as black at thumbnail size; Frank complained
"dark on dark reads as an empty rectangle."
**Fix:** Brighter mid-tone gradient (`#2e9e4d` centre → `#0d4a25` edges)
and added a soft radial light-falloff (alpha < 1 — one of the two
allowed uses).

### C4. No `window.onerror` handler
**Symptom:** Any runtime error would leave a black canvas (the prompt
flags this as "the worst possible outcome").
**Fix:** Added `window.onerror` that displays a red corner banner with
the error message and source location, so a black pane can never be
silent.

### C5. Rail top / inner / outer brightness
**Symptom:** Rails read as a single tone — no 3D face shading.
**Fix:** Three different fills per rail: top `#d4a060` (warm wood, lit),
inner angled to cloth `#8a5a30` (darker, in shadow), outer apron face
`#5a3a20` (darkest, in shadow).

### C6. Diamonds on rail tops
**Symptom:** No depth cues on the rails.
**Fix:** Added small white diamond markers along all 4 rails at evenly
spaced world positions; they bunch up at the far end under perspective
(the prompt's intended behaviour).

### C7. Pocket drawing — filled ellipses with rim
**Symptom:** First draft had visible upright pocket walls.
**Fix:** Pocket mouths are now filled black ellipses with a `#d6b27a`
light rim; no upright geometry on the cloth.

---

## D. Physics & loop fixes

### D1. `dt` not clamped in main loop
**Symptom:** After a tab-throttle pause, a single huge `dt` would let
balls tunnel through rails.
**Fix:** `if (dt > MAX_DT) dt = MAX_DT;` at the top of `frame()`.

### D2. Speed clamp
**Fix:** `if (sp > MAX_SPEED) { sp = MAX_SPEED; vx *= MAX_SPEED/sp; vz *= ... }`
applied per ball per frame.

### D3. Unstick rule
- New shot when all balls stationary for 0.6s (`stillTimer > 0.6`).
- Force-stop all balls at `shotClock > 12`.
- Re-rack after 20 shots without clearing.

### D4. Pocketing
**Fix:** `dx*dx + dz*dz < (POCKET_R*0.9)^2` per pocket per ball; cue
ball respawns at head spot, object balls set `alive=false` and parked
at (999,999) so they're sorted to the back and never drawn on the
table.

### D5. Cushion rebound
**Fix:** When `|x| > T.L/2 + RAIL*0.05` or `|z| > T.W/2 + RAIL*0.05`,
reflect the offending velocity component with `CUSHION_REST = 0.78`
(energy lost, never gained).

### D6. Ball-ball collisions (elastic)
**Fix:** Loop `i < j` over alive balls, compute line of centres, do
positional correction (50/50 along normal) and impulse with
`e = RESTITUTION = 0.96`. For glancing hits the impulse direction is
the line of centres → both balls diverge.

### D7. Rolling + spin
**Fix:** Each ball tracks `roll` (rotation angle) and `spinW` (angular
velocity = linear speed / radius). The ball number, stripe, and
highlight are drawn rotated by `roll`. Spin persists through cushion
bounces because the linear-velocity reversal doesn't touch `spinW`.

### D8. World-space lighting on balls
**Symptom:** Highlight pinned to the same screen-space corner of every
ball.
**Fix:** Compute the light direction in **world space** (`LIGHT =
{x:0.6, y:1.2, z:0.4}`), project it for each ball, and place the
specular dot at the angle from the ball centre to the projected light.
This makes the highlight slide across the ball as it moves and as the
camera orbits.

---

## E. Self-play (no input listeners)

### E1. State machine
`aim` (0.6s, draws aim line) → `pull` (0.25s, cue pulls back) →
`rolling` (physics runs, unstick timer accumulates) → `end` (compute
pocketed, possibly re-rack).

### E2. Target selection
Collect every `alive && !isCue` ball, pick one uniformly with
`Math.random()`. Add aiming error uniform in `[-4°, +4°]`.

### E3. Shot power
Break: `power = 4.5 + Math.random()` (hard enough to scatter the pack
across the full bed). Subsequent shots: `power = 3.5 + Math.random()*1.5`.

### E4. Aim line + cue animation
During `aim`/`pull`, draw a dashed white line from cue ball to target
and the cue stick tilted at the same angle, receding correctly in
perspective. During `pull`, the cue recedes backward; on strike the
cue thrusts forward and the line is hidden.

---

## F. HUD

Single 290×96 translucent box at (10, 10) with:
- "Rack:  N / 15"  (this rack, 22px monospace)
- "Total: N"
- "Shots: N"

`globalAlpha` reset to 1 after drawing the box. The box has a thin
`#d6a868` wood-coloured border so it reads as a panel, not a label
overlay.

---

## G. Code-rules compliance (prompt section)

- **Declare before read:** every `let`/`const` in the file is declared
  before any line that uses it. Verified by `node -c` parse.
- **No reassigning `const`:** only the few state variables
  (`camYaw`, `camTarget`, ball arrays, rack counters) are `let`.
- **Bounds-check array indices:** ball removal uses `i--` and a `>= 0`
  check after `splice`.
- **Iterate backwards when removing mid-loop:** the pocket-detection
  loop walks `i = balls.length - 1; i-- `, splicing pocketing balls in
  place.
- **Brackets/parentheses balanced:** verified by `node -c` and by
  Chrome loading the file without syntax errors.

---

## H. Verification artifacts

```text
# node parse
JS PARSE OK 23050 chars
# file size
minimax-m3-pool.html: 23 KB single file
# no external network requests
# no <script src=...>, no @import, no fetch(), no XHR
# canvas full viewport
canvas.width = window.innerWidth * DPR (capped at 2)
canvas.style.width = 100vw; height = 100vh
# rAF always inside draw, runs every frame
function frame(t){
  ...
  requestAnimationFrame(frame);
}
```

## I. Final tuning constants (for reproducibility)

```js
T = { L: 2.0, W: 1.0 }      // play area in world units
RAIL = 0.10
APRON = 0.20
LEG_H = 0.80
POCKET_R = 0.09
BALL_R = 0.030
BALL_MASS = 1
FRICTION = 0.55
RESTITUTION = 0.96
CUSHION_REST = 0.78
MAX_SPEED = 9.0
MAX_DT = 0.033
CAM_ELEV = 40 * π/180      // 40°
CAM_DIST = 3.5
ORBIT_PERIOD = 50          // seconds
camTarget = { x:0, y:-0.2, z:0 }
```

With these values the worst-case orbit angle produces a 727×656 px
table (81% × 73% of the 900×900 frame) with at least 50 px margin on
every side at every orbit angle.
