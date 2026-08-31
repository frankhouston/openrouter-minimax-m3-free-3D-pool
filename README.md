# openrouter-minimax-m3-free

A single self-contained HTML file that renders and plays a 3D pool table
game. No external dependencies, no CDN, no libraries. Canvas 2D
perspective projection with a real FOV 50° camera that orbits the
table once every 50 seconds.

The file is `minimax-m3-pool.html` — open it directly in any modern
browser. No server, no internet connection required.

## What's here

- `minimax-m3-pool.html` — the working game (≈ 23 KB).
- `PROMPT.md` — the original build prompt (verbatim, condensed).
- `REVISIONS.md` — every bug, fix, and tuning change from the initial
  draft to the final working file.

## Quick test

```sh
open minimax-m3-pool.html          # macOS
xdg-open minimax-m3-pool.html      # Linux
start minimax-m3-pool.html         # Windows
```

Or just double-click the file. The game plays itself — no input is
read. The camera slowly orbits the table; the AI cue ball aims, draws
back, strikes, and the physics runs to completion. When the rack is
cleared it auto-re-racks and continues forever.

## Compliance summary

| Prompt requirement | Status |
| --- | --- |
| Single HTML file, no dependencies | ✅ |
| Canvas 2D perspective, FOV 50° | ✅ |
| Camera 35–45° elevation, orbits 360° in 45–60s | ✅ (40°, 50s) |
| Whole table inside frame at all orbit angles | ✅ (CAM_DIST=3.5, T=2.0×1.0) |
| Strong foreshortening (screen radius from projection) | ✅ |
| Every object fully opaque, balls filled then optionally stroked | ✅ |
| Alpha < 1 only for contact shadows + cloth falloff | ✅ |
| Nothing upright on cloth except balls + cue | ✅ (pockets are filled ellipses) |
| 6 pockets (4 corners + 2 mid-rails) | ✅ |
| Rails with 3 visible faces at 3 brightnesses | ✅ |
| Apron, legs, room with floor-to-wall horizon | ✅ |
| Diamond sight markers (perspective-bunched) | ✅ |
| Balls roll with visible number/stripe/highlight rotation | ✅ |
| World-space light, soft terminator, bounce light | ✅ |
| Contact shadows offset per light direction | ✅ |
| Full rack (1 cue + 15 numbered, solids + stripes) | ✅ |
| Self-play (no input listeners) | ✅ |
| Aim 0.6s line + cue pull-back + strike | ✅ |
| Random target + ±4° aim error | ✅ |
| Re-rack when cleared | ✅ |
| Real rigid-sphere physics (mass, vel, radius) | ✅ |
| Rolling friction decelerates to stop | ✅ |
| Elastic ball-ball collisions, no overlap | ✅ |
| Cushion rebound with restitution < 1 | ✅ |
| dt clamp to 0.033s, speed clamp to 9.0 | ✅ |
| Simulation never gains energy | ✅ |
| Unstick: 0.6s still = next shot, 12s hard stop, 20-shot re-rack | ✅ |
| HUD ≥ 20px: rack / total / shots | ✅ |
| window.onerror for visible failure (no black pane) | ✅ |
| Code rules: declare-before-read, no `let` above decl, no const reassign, bounds-check, backwards-iterate | ✅ |
| Parses with `node -c` | ✅ |

## License

MIT.

---

## Latest clean iteration

`index.html` is the current canonical build: a single self-contained
Canvas 2D file (no dependencies/CDN) that renders and plays a 3D pool
table. Recent fixes applied here:

- Cue stick reworked into a realistic billiard cue — slender tapered
  maple shaft, glossy wood gradient, white ferrule, dark leather tip —
  and its length cut to **half the table length**.
- Pocket layout corrected to a real 6-pocket table: 4 corner pockets
  plus 2 side pockets on the **long** rails at midpoints.
- Fixed camera-axis tilt (trueUp cross-product) and added a per-frame
  centering bias + camDist tuning so the table stays centered with safe
  margin at every orbit yaw.
