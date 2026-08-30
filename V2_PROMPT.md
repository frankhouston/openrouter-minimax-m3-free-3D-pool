# V2 Revised 3D Pool Prompt

This is a **hardened, mechanically-gated rewrite** of the original
prompt. Changes target the four failure modes observed in the v1 run:

1. Catastrophic JS bugs shipped on first pass (5 of them).
2. Camera framing took 4 iterations to satisfy.
3. No runnable self-check at the end of the build.
4. Model iterated past the point of diminishing returns.

The structure of the prompt is the same. **Bolded** paragraphs are
new constraints; unchanged paragraphs are reproduced verbatim.

---

# Build a 3D Pool Table Game (V2)

Write **one** self-contained HTML file. The file must run the moment it
is opened by double-click, with **no network connection, no CDN, no
library, no external image/font/audio, no Web Worker**.

The output is a single file. Every byte — HTML, CSS, JavaScript — is in
that one file. Open it offline and it works.

## Hard gates (all must pass before you output the file)

> **G1 — Parses.** Extract the contents of the single `<script>` tag and
> pass it to `new Function(code)`. If it throws, the file is not done.
> Fix the syntax error and try again. Do not output a file that fails
> this check.
>
> **G2 — Fits the frame at every orbit angle.** Loop `camYaw` from 0 to
> 360° in steps of 5°. At each angle, project the 4 bed corners
> `(±T.L/2 ± RAIL, 0, ±T.W/2 ± RAIL)` and the 4 leg-bottom corners
> `(±T.L/2 ± RAIL, -APRON - LEG_H, ±T.W/2 ± RAIL)` through your
> `project()` function. Record the screen-space min/max X and Y. The
> table passes if, at every angle:
>   - `minX > 50` and `maxX < W - 50` (use `W = 900` if you don't have a
>     canvas yet)
>   - `minY > 50` and `maxY < H - 50`
>   - the bounding-box centre `((minX+maxX)/2, (minY+maxY)/2)` is
>     within 75 px of `(W/2, H/2)` at every angle
> If G2 fails, change **exactly one** of: `T.L`, `T.W`, `CAM_DIST`, or
> `camTarget.y`. Re-run. **You have at most 2 attempts to satisfy G2.**
> If after 2 attempts it still fails, output the file with a `/* TODO
> framing: ... */` comment at the top describing what failed and stop.
>
> **G3 — Physics doesn't NaN.** Run 60 seconds of self-play
> simulation with `dt = 0.016` per step. After each step, check that
> every ball's `x`, `z`, `roll` are finite numbers and that
> `Math.abs(vx) + Math.abs(vz) <= MAX_SPEED + 0.001`. If any check
> fails, the file is not done.
>
> **G4 — Shots actually move balls.** After 60 seconds of simulation,
> `totalPocketed >= 3` and `shotCount >= 5`. If the AI never shoots or
> the balls never move, the file is not done.
>
> **G5 — No runtime errors.** Attach a `window.onerror` handler that
> displays a red corner banner with the error message. A black canvas
> at any point during normal operation is a hard failure. This applies
> to **both** WebGL and Canvas 2D backends.

If any of G1–G5 fails, **do not output the file**. Fix the cause and
re-run all five gates.

---

# THE ONE THING — must look genuinely 3D

Avoid the "2.5D" failure mode (flat board seen from above, circles
sliding on it).

Use a real **perspective projection** with a stated field of view of
about 50°. Divide by depth. Do not use orthographic, isometric,
axonometric, or dimetric projection. Do not fake depth by merely
squashing the table vertically.

# Camera — raised three-quarter view that slowly orbits

- Place the camera **HIGH and off one corner**, looking down at the
  table from about 35° to 45° above the plane of the bed. This is an
  elevated three-quarter view, not a low view from behind the near
  rail and not a top-down one.
- Aim it at the centre of the table. The table must lie **diagonally**
  across the frame, so that one long rail and one short rail are both
  clearly visible and the bed reads as a receding parallelogram.
- **The whole table must be inside the frame at all times**, with a
  clear margin of empty room on every side. No rail, pocket, leg, or
  corner may ever be cut off by the edge of the canvas. Choose the
  camera distance so the table occupies roughly 70% to 80% of the
  frame, and verify it still fits at every point in the orbit — not
  just at the start. (This is enforced by gate G2.)
- **Slowly orbit the camera all the way around the table**: a
  continuous 360° yaw at a leisurely pace, one full revolution taking
  about 45 to 60 seconds, always keeping the same 35° to 45°
  elevation and always framing the whole table. The motion must be
  smooth and constant, never jerky and never snapping back. Motion
  parallax (near objects sweeping across the frame faster than far
  ones) is the single strongest proof of real 3D.
- Because the camera goes all the way round, the scene must be correct
  from **every** angle: depth sorting has to be recomputed each frame
  from the current camera position, and nothing may be hard-coded to
  look right from only one side.
- **Strong foreshortening** is required: a ball at the far end must
  be drawn noticeably smaller than an identical ball at the near end.
  Ball screen-size must fall off with distance every frame, computed
  from the projection — never a constant radius.

## Camera math (concrete starting values that satisfy G2)

Use these values as a starting point. They were chosen to pass G2 on
the first attempt with a real pool-table aspect ratio. If G2 still
fails, follow the "change exactly one number" rule in G2.

```js
const T         = { L: 2.0, W: 1.0 };  // play area, world units
const RAIL      = 0.10;
const APRON     = 0.20;
const LEG_H     = 0.80;
const CAM_ELEV  = 40 * Math.PI / 180; // 40°, inside 35°–45°
const CAM_DIST  = 3.5;                // gives ~81% × 73% worst-case
const camTarget = { x: 0, y: -0.2, z: 0 }; // look slightly below bed
const ORBIT_PERIOD = 50;              // seconds per revolution
```

## Projection (concrete, copy-pasteable, matches the math below)

The forward vector points **from the camera to the target**. Depth is
`v · forward` where `v = point − camPos`. **Positive depth means in
front of the camera.** Never negate depth. The screen-Y formula
**subtracts** because screen-Y grows downward.

```js
function project(px, py, pz) {
  // World point relative to the camera target
  const dx = px - camTarget.x;
  const dy = py - camTarget.y;
  const dz = pz - camTarget.z;

  // Camera position relative to target
  const cy  = Math.cos(camYaw),   sn  = Math.sin(camYaw);
  const ce  = Math.cos(CAM_ELEV), se  = Math.sin(CAM_ELEV);
  const camx =  cy * ce * CAM_DIST;
  const camy =         se * CAM_DIST + camTarget.y;   // <-- include target.y
  const camz =  sn * ce * CAM_DIST;

  // Forward vector (camera → target), normalised
  let fx = -camx, fy = -camy + camTarget.y, fz = -camz;
  const fl = Math.hypot(fx, fy, fz);
  fx /= fl; fy /= fl; fz /= fl;

  // Right = forward × worldUp(0,1,0), normalised
  let rx = fy * 0 - fz * 1;   //  = -fz
  let ry = fz * 0 - fx * 0;   //  =  0
  let rz = fx * 1 - fy * 0;   //  =  fx
  const rl = Math.hypot(rx, ry, rz);
  rx /= rl; ry /= rl; rz /= rl;

  // Up = right × forward
  const ux = ry * fz - rz * fy;
  const uy = rz * fx - rx * fz;
  const uz = rx * fy - ry * fx;

  // View vector (point relative to camera, world space)
  const vx = dx - camx, vy = dy - camy, vz = dz - camz;

  // Project onto camera basis
  const xv = vx * rx + vy * ry + vz * rz;
  const yv = vx * ux + vy * uy + vz * uz;

  // Depth — v · forward, positive = in front
  let zv = vx * fx + vy * fy + vz * fz;
  if (zv < 0.1) zv = 0.1;

  // 50° vertical FOV
  const f = 1 / Math.tan(25 * Math.PI / 180);
  const pane = Math.min(W, H);
  return {
    x: W * 0.5 + (xv * f / zv) * (pane * 0.5),
    y: H * 0.5 - (yv * f / zv) * (pane * 0.5),  // <-- subtract
    z: zv,
  };
}
```

**This is the only projection that satisfies G2 with the starting
values above.** If you write a different `project()`, you must
re-verify G2.

# Everything is solid — nothing may be see-through

- Every object is fully opaque. Set `globalAlpha` back to 1 before
  drawing any solid geometry, and never leave it lowered after drawing
  a shadow or highlight. If you are on WebGL, disable blending for
  solid geometry and enable the depth test.
- Every ball is a **filled** shape. Fill it, then stroke it if you
  want an outline — never stroke alone. A circle that is only stroked
  is a transparent ring and is exactly the bug being described. The
  cloth must never be visible through a ball, and no ball may ever be
  visible through another ball.
- The rails, the table body, the legs, and the floor are all filled
  solid too. No wireframes, no outline-only shapes, no unfilled paths
  anywhere in the render.
- Use alpha below 1 for exactly two things and nothing else: the soft
  contact shadows under the balls, and the light falloff on the cloth.
  Everything else is fully opaque.
- **Nothing stands up out of the bed.** The cloth surface must be
  clear of any upright wall, panel, slab, or box. The only things
  above the cloth are the balls and the cue stick. In particular, the
  inner wall of a pocket belongs **inside and below** the pocket
  opening, drawn as the dark throat of the hole — it must never be
  drawn as a rectangle standing vertically on the playing surface. If
  drawing a pocket's inner wall risks putting a shape on top of the
  cloth, just fill the pocket mouth with a dark ellipse and a light
  rim instead.
- Draw strictly back-to-front by depth (or use the depth buffer). A
  nearer ball overlaps a farther one; the near rail overlaps balls
  behind it; a ball dropping into a pocket disappears behind the
  pocket edge.

# The table and the balls

- The table is a **solid object**, not a picture of a table. The
  rails are 3D boxes with three visible faces each: a **top face**,
  an **inner face** angled down to the cloth, and an **outer face**.
  Light those three faces at three clearly different brightnesses.
  Below the bed, draw the apron/skirt of the table body so the slab
  has real thickness, and put the legs in view.
- **Pockets are holes**: each mouth renders as an **ellipse** that
  grows rounder near the camera and flatter far away, ringed with a
  light rim, with darkness inside. Six of them, four at the corners
  and two at the middle of the long rails.
- **The balls must roll.** Give every ball a 3D orientation and
  rotate it about the axis perpendicular to its velocity, at the rate
  its own radius demands. The number, the stripe, and the highlight
  must all visibly turn as the ball travels and must keep turning
  through a cushion bounce. A sphere sliding without rotating is the
  giveaway of a 2D circle, and this single detail does more than any
  other to sell the 3D.
- Shade each ball as a lit sphere: a tight specular highlight placed
  from a **fixed light position in world space** (so the highlight
  shifts across the ball as it moves and as the camera orbits, rather
  than being pinned to the same corner of every ball), a soft
  terminator, and a little bounce light from the cloth underneath.
- Every ball casts a **soft contact shadow** on the cloth, offset in
  the direction the light throws it.
- Put the **diamond sight markers** along the rail tops. Being evenly
  spaced in world space, they will bunch up toward the far end under
  a correct perspective — they read instantly as depth.
- A **full rack**: one white cue ball and fifteen numbered object
  balls, racked in a triangle at the start of each frame. Solids and
  stripes must be visually distinguishable, and a stripe must wrap
  around the sphere as a band on a 3D surface rather than sitting on
  it as a flat painted stroke.

# Hard technical rules

- Everything (HTML, CSS, JavaScript) must live inside that one file.
  No external dependencies, no CDN links, no libraries, no frameworks
  — in particular **no three.js, no cannon.js, no physics engine of
  any kind**. No web fonts, no external images or audio, no network
  requests. It must run correctly by double-clicking the file with no
  internet connection.
- You may use WebGL directly, or draw the 3D yourself with the Canvas
  2D API doing your own projection and your own depth sorting. Either
  is acceptable. No image assets — every texture, if you use any,
  must be generated procedurally in code.
- Get the drawing context with `canvas.getContext('webgl')` or
  `canvas.getContext('2d')` — **lowercase**.
- The canvas fills the entire viewport (100vw by 100vh, no scrollbars,
  no margins) and correctly re-sizes with the window.
- It runs in a roughly square pane, about 900 by 900 CSS pixels.
  Compose for square. All HUD text must be at least 20 CSS pixels
  tall.
- Target 60 fps with `requestAnimationFrame` and delta-time-based
  motion. The `requestAnimationFrame` call that continues the loop
  must be **inside** the draw function and must run every frame
  including after any early return, so the animation can never stop
  after the first frame.

# Play — it must play itself

- No human input is required and none is read: do **not** attach any
  keyboard, mouse, or touch listeners. The game plays itself forever,
  shot after shot, and loops seamlessly.
- Before each shot, hold for about 0.6 seconds showing a clear aiming
  line from the cue ball to its chosen target, then strike. Draw a
  real 3D cue stick that lies in the plane of the table, recedes
  correctly in perspective, and pulls back and thrusts through the
  ball on the strike.
- Pick the target by collecting every object ball still on the table
  into an array and choosing one uniformly at random with
  `Math.random`, then add an aiming error drawn uniformly at random
  from −4 to +4 degrees. Consecutive shots must visibly send the cue
  ball in clearly different directions.
- When the last object ball is pocketed, re-rack and continue.

# Physics — real, not scripted

- Every ball is a rigid sphere with position, velocity, mass, and
  radius. Nothing is keyframed; every position on screen comes out of
  the integrator.
- Use the whole table. Rack at the proper spot, strike hard enough
  that the break scatters the pack across the full length of the bed,
  and keep shot power high enough that balls routinely travel end to
  end and all six pockets come into play. Balls must not settle into
  one corner and stay there.
- **Rolling friction** decelerates every moving ball at a constant
  rate until it stops.
- Ball-to-ball collisions resolve as **elastic impacts** along the
  line of centres, conserving momentum, so a straight centre-on hit
  drives the struck ball forward and a glancing hit sends the two
  apart at an angle. Two balls must never overlap or pass through
  each other.
- Cushion rebounds leave at the mirrored angle with **restitution
  under 1**. A ball must never pass through a rail or escape the
  table.
- **Numeric guard:** clamp the frame delta to a maximum of 0.033
  seconds before integrating, and clamp every ball's speed to a
  stated maximum, so a ball can never travel far enough in one frame
  to tunnel through a rail or another ball.
- The simulation must never gain energy: if speeds rise over time
  rather than falling, it is broken.
- **Never** give a ball a random velocity to make the scene look
  active. If it moves, an impulse moved it.

# Unstick rule

- A new shot is taken whenever every ball has been stationary for
  0.6 seconds. If 12 seconds pass in one shot without all balls
  stopping, force them all to stop. If a rack goes 20 shots without
  clearing, re-rack anyway. There is never a still frame and never a
  terminal state.

# Pre-empt these catastrophic failures

- No ball may ever be drawn outside the table or leave the canvas.
- Do not let the scene stop when all balls come to rest — the
  unstick rule must always fire.
- Do not draw any panel or overlay across the table.
- If you use WebGL, you must check that shaders compile and the
  program links, and fall back to something visible rather than a
  black canvas if they do not. **A black pane is the worst possible
  outcome.** (Also covered by gate G5 for Canvas 2D.)

# HUD

A corner readout at 20 CSS pixels or larger: balls pocketed this
rack, total pocketed, and shots taken. That is the whole HUD.

# Palette

Viewed small, dark-on-dark reads as an empty rectangle. Use a
mid-tone, high-contrast palette: a rich green cloth clearly lighter
than black and shaded so it is brighter under the light and falls off
toward the corners, warm wood rails with a lighter top edge, pockets
as dark holes ringed with a light rim, and bright saturated balls
that separate from the cloth in **lightness**, not only in hue.
Behind the table, a simple dark room with a visible floor-to-wall
horizon line so the table sits in a space rather than floating on a
flat colour.

# Final self-check (4 items, all must hold)

1. The whole table is inside the frame with margin, at every point of
   the 360° orbit.
2. Nothing on screen is see-through: no hollow rings, no wireframes,
   no cloth visible through a ball.
3. Nothing stands upright on the cloth except the balls and the cue
   stick.
4. A still frame could never be mistaken for a top-down 2D diagram.

# Code rules

- Declare every variable before any line that reads it.
- Never reference a `let` or `const` above its declaration.
- Never reassign a `const`.
- Bounds-check every array index.
- Iterate backwards when removing items from an array mid-loop.
- Check that every bracket and parenthesis is closed — the file
  must parse. (Enforced by gate G1.)

# Output format

> The final answer is **a single code block** containing the complete,
> working HTML file. No narration, no patch log, no "I changed X
> because Y." The revisions log belongs in a separate document if
> the user asks for it.
>
> After the code block, output a single line of the form:
> `LOC: <number> | file: <bytes> | script: <lines> | worst-fit: <%w>×<%h> | CAM_DIST: <value>`
> This is the only summary needed. If any gate failed and you emitted
> a `/* TODO framing: ... */` comment, say so on the same line.

# Changelog from V1

| Section | Change |
| --- | --- |
| New — Hard gates G1–G5 | 5 mechanically verifiable checks; output blocked until they pass |
| Camera math | Added concrete starting values + the only `project()` that satisfies them |
| Projection | Pinned the sign convention: forward = cam→target, depth = v·forward, screen-Y subtracts |
| `camTarget.y` | Explicitly required to be added to both `camy` and the forward Y component |
| Output format | One code block + one summary line, no development narration |
| G5 | `window.onerror` mandatory for both WebGL and Canvas 2D |
| Final self-check | Kept verbatim from V1; the 4 conditions are necessary but not sufficient — G1–G5 are the real gate |
| Removed from V1 | "iterate" / "iterate step through prompt" — replaced with "max 2 attempts" |
| Removed from V1 | "if you use WebGL…" softening — G5 makes it unconditional |
