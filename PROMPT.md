# Original Prompt (verbatim, condensed)

Build **ONE single self-contained HTML file** containing a playable 3D pool table
game that runs the moment the file is opened. No CDN, no libraries, no
three.js, no cannon.js, no external images/audio/fonts. Canvas 2D or WebGL.
Canvas fills the viewport (100vw × 100vh), ~900×900 square, HUD ≥ 20px, 60fps
via `requestAnimationFrame` with delta-time motion. The rAF call must be
**inside** the draw function and run every frame (including after early
return).

## THE ONE THING — must look genuinely 3D
Real perspective projection, FOV ~50°, divide by depth. No orthographic /
isometric / axonometric / dimetric. No vertical squash faking depth.

## Camera — high three-quarter view that slowly orbits
- Camera HIGH and off one corner, 35–45° above bed plane.
- Aim at table centre; bed reads as a receding parallelogram with one long
  rail and one short rail visible.
- **Whole table inside frame at all times with margin on every side**.
  70–80% of frame.
- Continuous 360° yaw orbit, 45–60s per revolution, smooth.
- Depth sort recomputed every frame from current camera position.
- **Strong foreshortening**: far balls smaller than near balls, screen-size
  derived from projection per frame, never constant radius.

## Solid, nothing see-through
- Every object fully opaque. `globalAlpha` back to 1 after shadows.
- Balls are **filled shapes**, then optionally stroked. Never stroke alone
  (hollow ring = bug).
- Rails, body, legs, floor all filled solid. No wireframes.
- Alpha < 1 only for: contact shadows under balls, cloth falloff.
- **Nothing stands up out of the bed** except balls and cue stick. Pocket
  inner wall must be the dark throat below the opening, never an upright
  slab on the cloth.
- Draw back-to-front by depth (or depth buffer).

## Table
- Rails: 3 visible faces each (top, inner angled down to cloth, outer) lit
  at 3 different brightnesses.
- Apron/skirt below bed with thickness; legs visible.
- **Six pockets**: ellipses (rounder near camera, flatter far), light rim,
  dark inside. 4 corners + 2 middle of long rails.
- Balls **roll** with 3D orientation. Number, stripe, highlight all visibly
  turn. Rotation persists through cushion bounces.
- Each ball shaded as a lit sphere: tight specular highlight from a fixed
  **world-space** light, soft terminator, bounce light from cloth below.
- Every ball casts a soft contact shadow on the cloth.
- Diamond sight markers along rail tops (bunch toward far end in
  perspective).
- Full rack: 1 cue + 15 numbered, solids + stripes distinguishable, stripe
  wraps as a band on a 3D surface.

## Technical
- `canvas.getContext('webgl')` or `canvas.getContext('2d')` — lowercase.
- All inline. No external dependencies.
- If WebGL: verify shader compile + program link, fall back if they fail
  (black canvas is the worst outcome).

## Play — game plays itself
- No input listeners (no keyboard/mouse/touch).
- Per shot: ~0.6s aim hold showing a clear 3D aim line, then strike.
- Cue stick: real 3D, lies in table plane, recedes correctly in perspective,
  pulls back and thrusts through the ball.
- Target: pick ONE random object ball, add aiming error uniform in
  [-4°, +4°]. Consecutive shots visibly diverge.
- When last object ball pocketed → re-rack.

## Physics — real, not scripted
- Every ball: position, velocity, mass, radius. Integrator-driven, not
  keyframed.
- Break scatters the pack across the full length of the bed. Power enough
  to reach end-to-end and use all 6 pockets.
- Rolling friction decelerates moving balls at constant rate.
- Ball-ball collisions: elastic along line of centres, momentum conserved.
  No overlap/passthrough.
- Cushion rebounds: mirrored angle, restitution < 1. No tunneling.
- **Numeric guards**: clamp frame `dt` to max `0.033s`; clamp every ball
  speed to a stated max so a ball can't tunnel.
- Simulation never gains energy.
- Motion only via impulse — never inject random velocity to fake activity.

## Unstick rule
- New shot when all balls stationary for 0.6s.
- If 12s pass in one shot → force stop all balls.
- If a rack goes 20 shots without clearing → re-rack anyway.
- Never a still frame, never a terminal state.

## Catastrophic failure pre-empts
- No ball ever drawn outside the table or leaves the canvas.
- Scene never stops at rest (unstick always fires).
- No panel/overlay across the table.
- WebGL: shader check + fallback if needed.

## HUD
Corner readout ≥ 20 CSS px: balls pocketed this rack, total pocketed, shots
taken. That's the whole HUD.

## Palette
Mid-tone, high-contrast. Rich green cloth clearly lighter than black,
shaded brighter under the light, falls off toward corners. Warm wood
rails with lighter top edge. Pockets: dark holes with light rim. Balls:
bright saturated, separate from cloth in **lightness** (not just hue).
Background: simple dark room with visible floor-to-wall horizon line.

## Self-check (4 must pass)
1. Whole table inside frame with margin at every orbit point.
2. Nothing see-through: no hollow rings, no wireframes, no cloth visible
   through a ball.
3. Nothing upright on cloth except balls and cue stick.
4. A still frame could never be mistaken for a top-down 2D diagram.

## Code rules
- Declare every variable before any line that reads it.
- Never reference `let`/`const` above its declaration.
- Never reassign a `const`.
- Bounds-check every array index.
- Iterate backwards when removing items from an array mid-loop.
- All brackets/parentheses closed; file must parse.
