# Changelog

All notable changes to cyrius-mine-cart are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/), and this project adheres to
[Semantic Versioning](https://semver.org/).

## [Unreleased]

**Forks — the build order's step 6.** Every few hundred metres the shaft splits: a sign in the tunnel
(design plate 1A) says **◄ WET ×2** and **DRY ×1 ►**, a double-tap throws the switch, the lit rails
change sides with it, and the cart commits at the points. The wet shaft pays double and runs on slick
rails between closer hazards; the spur is dry and pays the ordinary rate. The ring still holds one
path — the other branch is only ever drawn, as a mirror — and the generator waits at the end of the
fork's neck until the cart has chosen. The new gate pass that rides the fork both ways at the extremes
found, the first time it ran, a 2× `|D|` margin that the ride path had never leaned far enough to see;
face-on geometry is now clipped to the draw rect, and the margin is 21×.

### Added

- **Forks** (`src/track.cyr`, `src/geom.cyr`). The generator lays a fork in stages measured from its
  points: a straightening lead-in, a sterile approach (`CFG_FORK_CLEAR` — nothing on the track, no side
  tracks), the sign `CFG_FORK_SIGN_LEAD` segments before the points, and a `CFG_BRANCH_NECK`-segment
  neck in which each branch turns away at up to `CFG_FORK_DC` and eases back to straight. The first
  fork's points are at segment `CFG_FORK_FIRST` (282 m); each branch runs 156–312 segments, its last
  `CFG_BRANCH_TAIL` laid as trunk again, and the next fork follows 60–120 segments on.
- **The switch.** `IN_SW_LEFT` / `IN_SW_RIGHT` (the double-tap that 0.2.1 decoded and nothing read)
  throw it any time before the points: left is the wet shaft, right the service spur, and a cart whose
  switch is never touched takes the spur (plate 1B: "NO INPUT — CART TAKES IT"). Crossing the points
  makes it final — in the position carry, not in `cart_tick`, so anything that moves the cart locks a
  fork and releases the generator.
- **One path in the ring, and a mirror.** The trunk under a fork is dead straight, so the branch not
  taken is exactly the negation of the neck's curvature. Throwing the switch negates the neck in place
  (`trk_fork_switch`): nothing the camera has seen moves, and on screen the lit rails and the lit panel
  change sides, as a turnout's blades would. The generator runs 48 segments ahead but stops at the end
  of the neck until the choice is made; the neck is longer than the view, so the pause never shows.
- **The split, drawn.** From the points to the **nose** (`CFG_FORK_NOSE`, where the two bores are
  170 ru apart against the 168 they need) both branches share one chamber whose walls follow each
  branch's outer side, so each track keeps a full bore of room on the side it can lean toward. The
  walk lands a slab endpoint exactly on the nose. Past it the ring's bore continues and the branch not
  taken is a dark mouth; the rock between them faces the camera. The unlit branch's rails are drawn
  from a new one-row atlas tile in dim steel, so the route the cart will take is the only bright one.
- **The sign** (plate 1A): two 16 × 13 atlas panels per branch, lit and unlit, lettered in the HUD's
  3 × 5 cell and hung above the eye so it cannot be mistaken for a beam. In view 1.26 s before the
  points at 103 km/h and 1.36 s at the redline; tracktest re-derives the brief's 1.2 s from the frustum.
- **The wet shaft and the service spur.** Decided 2026-09-24: the wet shaft's "RISK III" is slick
  rails *and* closer hazards. Grip on `FLAG_WET` segments is scaled by `CFG_WET_GRIP` (0.80) after the
  speed fade, which moves the redline let-go curve from ~536 to ~430 brads/segment; hazards come every
  10–24 segments with 45% stacked pairs; treasure scores `CFG_WET_YIELD` (×2). The spur has its own
  knobs, set to the trunk's for now.
- **Face-on geometry is clipped to the draw rect** (`mc_fo_*` in `src/emit.cyr`). For a triangle at one
  `w`, `|D|` is its whole area times `65536/w`, on screen or off. Unclipped, the sign passed under at
  w 280 measured 193,130,496 and the dark mouth, with the camera jammed toward it as the nose reached
  the near plane, 819,686,400 — a 2× margin. Such surfaces are now convex polygons, clipped one rect
  edge at a time and fanned: exact, because at one `w` the mapping is affine and the clipped corners
  lie on whole-pixel edges.
- **The gate rides the fork both ways at the extremes**: to the first fork twice (switch thrown, switch
  left alone), through the approach, chamber and nose every third tick with the camera on the rail and
  jammed to either wall, at all three consoles — 1,440 frames. And a face-on corner refused for want
  of room counts as a pixel-losing drop.
- **The input log records which simulation made it** (the u16 that 0.2.2 wrote as a reserved zero).
  A log is only a seed and words, so a build that simulates differently replays a different ride and
  reports a divergence that reads exactly like a determinism bug. `--replay` and `film --log` now
  refuse another revision's log by name. This build is revision 1; 0.2.2 is 0.
- **`--verify --pilot 1 --frames N`** verifies frame N of the autopilot's ride — the frame `shot` and
  `film` call N. The default, frame 120 of the hands-off ride, never reaches a fork; 715 is inside one.
- `--sim` reports each fork taken and the ticks spent on the wet shaft, and slipping there;
  `tools/shot.cyr --pilot 2` rides the autopilot with every switch left alone; shot captions name the
  points, the switch, the sign, the neck and the wet shaft; `--help` documents the switch.
- Tests: the segment ring 43 → 84 assertions (the fork's placement, straightness, sterility and
  profile; the pause, the lock and the refusal after it; the mirror; the wet flag; determinism across
  a choice; and the nose, the 1.2 s telegraph and the neck-outlasts-the-view re-derived from the
  constants), cart + input 110 → 124 (the switch word, the default, slick rails, the yield), replay
  10 → 14 (a switch left unthrown must diverge; another revision's log is refused by name).
- `docs/screenshots/fork.png`.

### Changed

- **The track differs from 0.2.2 from the first fork's lead-in on (segment 90, 230 m).** Everything
  before it is identical — the fork logic draws nothing from the rng until it is laid — so hands off,
  the cart still falls into the first gap at 87 m every time.
- The reference autopilot throws every fork's switch to the wet shaft, so `--sim`, the gate and the
  film all measure what the risky line demands.
- The replay's per-tick state hash includes the fork (which, which way, whether final, what is being
  laid).
- Corrected the window notes in the README, `src/main.cyr` and `src/gpu.cyr`: the transport is
  settled — setu 0.8.0 speaks the agnos channel band (`#97 chan_op`, of which `anu` was only a
  candidate name), and puka has presented a composited window since 2026-08-07. What still rules out a
  window is that no colour-writing op lets the caller name its destination.

### Removed

- `vendor/setu.cyr`: nothing included it, it carried the TCP-on-loopback transport agnos retired, and
  agnos's transport cut (`planning/ipc.md` §10.2) listed it as a hand-vendored copy to remove by hand.

### Notes

- Peak `|D|` 80,688,960 → **99,385,920** (margin 26× → **21×**), now measured over the fork at the
  extremes at all three consoles as well as everything 0.2.2 measured. Before face-on clipping, the
  same passes measured 819,686,400.
- Triangles per frame on the ride path 165–235 of 256 (0.2.2: 165–231, on a track without forks);
  through the fork at the extremes, at most 213. No drop that loses pixels in 2,520 frames.
- `--sim`, 30 s: the autopilot throws the first fork's switch at 281 m and reaches 732 m with no
  deaths; over two minutes it takes three wet shafts, reaches 3,031 m on one life, and slips for 13.6%
  of its 4,039 ticks on the wet shaft against 8.3% off it. Hands off: 87 m, every time, no fork.
- Binary 409,728 → 443,072 bytes (agnos: 413,656 → 451,088).
- **None of this has been on iron** — nor has 0.2.2. The roadmap lists what to run on the hardware.

## [0.2.2] — 2026-09-24

**Audit, repair, and the screenshots.** The toolchain moves to 6.6.6 with the kernel; the host gate's
independent re-parse turns out to have been checking the wrong corners, and is fixed and made to
prove it; and then the frame was *looked at* — a tool now writes screenshots on a host — which found
three things every gate had passed: a black wedge where the far roof should be, three-track runs that
were never on screen, and treasure that never rose above row 383 of 400. Fixing those found a fourth,
worse one: the cart on screen was 9.9 m ahead of the cart the rules were applied to. With the frame
honest, the build order's step 11 (the input-log replay) and a HUD land on top.

### Changed

- **Toolchain `6.6.2` → `6.6.6`**, the pin agnos 1.57.7 builds with, and the vendored stdlib re-synced
  from it: every file in `lib/` is byte-identical to 6.6.6's, plus the three modules 6.6.6's `io.cyr`
  now pulls in (`fmt`, `vec`, `alloc_cx`). That removes both `undefined function` warnings
  (`fmt_int`, `fmt_int_buf`) the 6.6.2 build printed — an undefined function compiles to a `ud2`
  stub, not a link error. Verified before any source change: the gate's output, three `--sim` runs
  and all 1800 frames of a 30 s film byte-identical across the two toolchains.
- **The binary is 23× smaller: 8,825,592 → 409,728 bytes** (agnos: 8,833,616 → 413,656). The `--wav`
  capture buffer (6.4 MB), the `--verify` readback buffer and the CPU reference's frame (1 MB each)
  were static arrays written into every binary on every target; they are heap now, allocated on the
  paths that use them.
- **The camera trails the cart by `CFG_CART_RZ` (124 ru, 9.9 m) instead of sitting on it.** The
  simulation's position is the cart's; the renderer had put the *camera* there and drawn the cart
  9.9 m further on, so the cart on screen reached a beam 0.37 s before the hit landed and had cleared
  the whole far lip of a gap before the simulation decided it was falling in. A player who jumped
  when the cart they could see reached the edge landed inside the gap. Renderer only: `--sim`, the
  replay hashes and the audio are unchanged by it.
- **The frame, redrawn from the atlas.** The texture's floor band is `u 0..31`, and `u 32..47` is now
  a sprite atlas painted from character bitmaps: the cart's rear plate (rim, hazard band, tail lamps,
  wheels), its core crate, the rider, a hazard-striped beam face, a faceted gem, a spark, and the
  HUD's font and swatches. The rails moved under the wheels (they were at ±7 ru, inside the cart's
  ±11). The floor is ballast only, as its comment already claimed while the code laid sleepers across
  the whole 13.4 m bore. The rock lost the joint-every-32 cm comb that read as planking and gained a
  timber set every segment, which sweeps past in step with the rail-joint click.
- **Beams hang at rider height** (30..42 ru below the eye: a standing rider's helmet reaches 38, a
  crouched one 45) with a hazard-striped face and a top, instead of at the arch's springing line
  6.9 m above the cart in the walls' own rock texture.
- **Things on the track are placed by segment, not by slab.** An object on the far half of a 64 ru
  slab was never drawn; now every beam and gem stands at the exact depth `cart_tick` resolves it, and
  each slab's floor is cut at the segment boundaries inside it, so a gap's lip no longer pops between
  slab boundaries as it approaches.
- `--sim`'s closing summary reports every life, not the last one — see Fixed.
- Source split: `src/emit.cyr` (the op `0x0F` ABI half of the old `geom.cyr`), `src/hud.cyr`,
  `src/replay.cyr`, and `src/gate.cyr` (the host gate, out of `main.cyr` so a test can run it).

### Added

- **Culling.** A triangle whose every vertex lies beyond one edge of the draw rect covers no pixel in
  any rasteriser and is no longer emitted. It is exact — the film is byte-identical with and without
  it — and it freed ~100 of every frame's 256 triangles, most of them floor, roof and wall slabs
  above, below and beside the screen. That budget is what the rest of this release draws with.
- **The input-log replay — the build order's step 11.** `--record F` (on the ride, or on `--sim`)
  writes every word handed to `cart_tick`, then a hash of the simulation state over every tick and a
  hash of the final frame's packed triangle list. `--replay F` re-runs it anywhere and exits 95 only
  if both reproduce; `film --log F` renders a recorded ride as video and checks the same two hashes.
  A ride that ends in a refused frame still writes its log. `--check` records a 40 s ride, replays
  it, compares the final frame as pixels, survives an intervening ride, refuses damaged logs, and
  flips one input bit to prove the replay notices (10 assertions).
- **A HUD, in the same record**: a depth odometer (top left), six integrity pips (top right) and a
  speed bar against the redline that turns amber above it (bottom left). Drawn last from the atlas,
  never shaken, and held in the budget reserve (`CFG_TRI_RESERVE` 16 → 32).
- **The cart shows what the simulation is doing**: it rises through the jump's arc, the rider drops
  below the rim while crouched, the inside wheel lifts and sparks spray off the outside one from
  `cart_warn() >= 2`, and the frame shakes as the derail clock runs down.
- Treasure is a faceted diamond in its lane, drawn at every depth, and stops being drawn once taken
  (`FLAG_TAKEN`, set by the simulation, read only by the renderer).
- **`tools/shot.cyr`**: screenshots (PPM) of the reference ride at any frames — frame N is `film`'s
  frame N — with a caption naming what the simulation has placed ahead. `--pilot 0` rides hands-off.
- **The gate re-parses the op record itself** — alignment, bounds, texture dimensions, triangle count
  and the kernel's `2^20` tile-triangle work budget (640×400 at 256 triangles spends 1,024,000 of it)
  — at the host's offset and on 800×600 and 2560×1440 consoles, from a header now packed by one
  function (`mc_pack_op`) the ride shares. It asserts that the emitter and the re-parse measured the
  **same** peak `|D|`, and plants two more mutations (a reserved dword, an op record off the tile) that
  must be caught and named.
- **`tests/deepvein.tcyr`**: `cyrius tests` runs the host gate.
- `docs/development/roadmap.md`, and `docs/screenshots/`.

### Fixed

- **The gate's re-parse evaluated `|D|` at the wrong corners.** The kernel bounds the denominator at
  the draw rect's corners in framebuffer-absolute space (`2*dx+1`); the emitter had been corrected to
  match when the frame moved to its on-screen offset, but the gate's independent re-parse still used
  a rect pinned at (0,0). Over the gate's own run it peaked at 290,364,480 where the kernel's corners
  give 277,334,400 — it was checking four points the kernel never evaluates. Restoring the old corners
  now turns the gate red.
- **A non-zero reserved dword was reported as code 21 (`GPO_E_WORK`)**; the kernel calls it 24
  (`GPO_E_TRILIST`).
- **The coverage check passed a hole in every frame.** Its floor was "more than half the screen", and
  0.2.1's frames — with the far roof undrawn — measured 82.6%. The top half (roof, arch and far wall:
  a gap never reaches it) must now be 99% covered, and was 100% on all 900 ride frames; the whole
  frame must be 75% (worst measured 87.6%, a gap under the cart). With 0.2.1's ceiling gate put back,
  the top half measures 82% and the gate fails.
- **The far ceiling was not drawn** (it was distance-gated to save budget), leaving op `0x0F`'s black
  background as a wedge across the top of every frame.
- **Three-track runs were never visible**: side tracks were gated to `rz < 112`, and the floor does
  not enter the frame until `rz ~115`. Treasure had the same gate and reached rows 383–399 of the
  frame in 15% of frames, never higher.
- `tools/dumptex.cyr` no longer compiled (ten undefined names since the constants moved to
  `config.cyr`); `build/dumptex` was a fossil nobody could regenerate.
- `--sim` counted every slipping tick of a life that ended twice, and its summary reported only the
  last life — the hands-off pilot's "0 falls" after seven falls into the same gap. `--sim --frames 0`
  died of SIGFPE. `cart_apex_hits` and `cart_derails` are session totals `cart_reset` never clears,
  and are now documented and reported as such.
- `--frames 9x` parsed as 162 and `--frames abc` as a negative (i.e. unbounded) ride; unknown flags
  were ignored, so `--chek` rode instead of gating. One parser (`mc_atoi`) now refuses both, and an
  unknown flag exits 2.
- The ride said "DERAILED" for a fall into a gap and for a cart destroyed by beams alike.
- A beam within 4 ru of the far plane pushed its top face's `w` past 4096 — caught by the gate the
  first time it ran (four `w`-band drops), fixed before it could reach iron.
- Stale comments in `config.cyr` (a 64 ru bore, an 8 ru gauge, rails at 45% of the threshold, a grip
  table computed from constants two retunes old) — recomputed from the current constants.

### Removed

- Dead code: `mc_ck_overlap` (measured overlap between the two records the frame no longer has),
  `mc_blit_fullscreen`, the unused `mc_slot_vb` / `mc_slot_tb`, the `CFG_SIDE_RZ` / `CFG_CHAMFER_RZ` /
  `CFG_BEAM_RZ` / `CFG_PICKUP_RZ` distance gates, `MC_TEX_CART`, and the per-endpoint `mc_s_seg`.
- `build/mine-cart-agnos` — a 0.1.0 artifact from 2026-07-30 that nothing builds or stages (the burn
  script stages `build/mine-cart_agnos`).

### Notes

- Peak `|D|` 277,334,400 → **80,688,960**: the ABI margin went from 7× to **26×**, because the worst
  `|D|` came from near-plane walls that were never on screen and are now culled.
- Triangles per frame 180–252 → **165–231** of 256, while drawing the whole roof, every side track,
  every beam, every gem, a five-part cart and the HUD. A 40,000-frame soak (autopilot and hands-off)
  peaked at 231 with no refused detail, no pixel-losing drop and no re-parse violation.
- `cyrius audit`: format clean, lint 51 warnings → 0, undocumented public functions 91 → 0, and a
  tests step that runs.
- **None of this has been on iron.** The README's screenshots are the CPU reference's; `--verify` on
  hardware is the only instrument that can see a frame that is legal and wrong. See the roadmap.

## [0.2.1] — 2026-09-11

⚠ **Recorded retroactively in 0.2.2.** This entry originally said only "no source change" beside the
toolchain bump, but four commits between 0.2.0 and it (`d4709b3`, `db002d1`, `941587e`, `52329b7`)
carried the build order's steps 5, 7 and 10 and were never written down. They are below.

### Added

- **Synthesised audio** (`src/audio.cyr`): a speed-tracking rumble through a Cytomic/Simper
  state-variable filter, rail-joint clicks at one per segment (so speed is audible), a brake squeal
  and a higher slip screech; whip and flood voices scaffolded. `--wav` renders 20 s to a file so a
  person can listen, since no assertion can say whether a mix sounds right. 14 assertions.
- **Grip.** The wheel flange holds the cart until the lateral force exceeds a capacity that falls with
  speed; past it the rail lets go — self-centring stops and three times the excess throws the cart
  outward — and only the brake can reach it.
- **Duck and jump.** Beams cost two of six integrity blocks unless crouched; three-segment gaps are
  death unless airborne. Hang time is fixed (26 ticks), so a braked cart comes up short; a crouch
  (27 ticks) cannot be cancelled. Hazards are placed with stacked pairs spaced wider than either
  commitment, and none before the cart can clear them.
- **Treasure** in three lanes, collected by lateral position at the segment crossing, scored by speed.
- **Three-track runs** (`FLAG_MULTI`): the bore widened to 84 ru and the lean threshold to 72 ru, with
  every track drawn as its own strip.
- **The cart, drawn** 124 ru ahead of the camera, sized from the gauge.
- The input word gained a double-tap SWITCH and a jump-plus-direction HOP, decoded and tested but not
  yet acted on by the simulation.
- `tools/film.cyr` — the ride as video, on a host — and one reference autopilot shared by `--sim`,
  `--wav`, the film and the gate.

### Changed

- **Toolchain `6.4.78` → `6.6.2`.** Build, tests and every bench/fuzz/distlib target re-verified at
  the new pin; the source was reformatted by `cyrfmt`, with no behavioural change.
- README and `main.cyr`: the setu transport correction of 2026-08-03 (TCP on loopback retired as the
  wrong primitive, pending the agnos socket).

## [0.2.0] — 2026-08-01

**DEEPVEIN.** The tunnel ride becomes a game. The world is no longer a closed-form sine wave — it is
a streaming ring of authored track segments — and the cart is no longer on rails in the figurative
sense: speed, brake, lean and derail are the whole loop, and the design brief's build order says
that loop has to be fun with nothing else in it before anything else is added. This release stops
exactly there, on purpose.

### Added

- **Fixed-point core** (`src/fx.cyr`). Q16.16 `fx_mul` / `fx_div` / `fx_lerp`, a 256-entry Q1.15
  sine table indexed by u16 brads (65536 = one turn, `cos = sin[i+64]` exact at every entry), and
  xorshift32. 65 assertions in `src/fxtest.cyr`, every expected value derived outside the program.
- **Segment ring + streaming generator** (`src/track.cyr`). 256 cm segments carrying curve, grade,
  kind, flags and branch indices as struct-of-arrays; 96 live in a 128-slot ring, 48 generated
  ahead, discarded behind. Position is `pos_seg` + `pos_sub` with a real carry. The generator emits
  *phrases* rather than per-segment noise, and is seeded per chapter so a death replays the
  identical track. 43 assertions.
- **Ring-driven renderer** (`src/geom.cyr`). `mc_build` walks an 8/16/32/64 render-unit slab ladder
  and turns per-segment curvature into a centre line by exact running integral. Arched cross-section
  (floor, walls, chamfers, ceiling) at 192 triangles a frame in ONE op `0x0F` record.
- **Cart physics** (`src/cart.cyr`). Lateral force is `curve * speed²`; the brake is fast and
  rebuilding speed is slow; lean past threshold for 30 consecutive ticks derails. Braking into a
  tightening bend and releasing at the apex pays a speed bonus — **96 km/h is the throttle ceiling
  and 103 km/h has to be earned.**
- **Input** (`src/input.cyr`). `#42` kbscan decoded to one input word per tick, handling all four
  silent hazards: tap-in-one-drain edges, the dangling `0xE0` prefix across drains, the six-byte
  `0xE1` Pause sequence, and a 256-byte drain so a dropped break code cannot stick a key.
- **`--sim`**, a headless tuning instrument. Runs the simulation with no GPU and no keyboard and
  prints telemetry for two scripted pilots — hands-off versus a crude autopilot. The GAP between
  them is the difficulty, and it is the only way to answer step 4's question on a host.
- **`src/config.cyr`**, one CONFIG block. Every tunable, every unit stated.

### Changed

- `MC_SEG` 60 → **32** render units. 60 ru is 480 cm and 480 mod 256 = 224, so a slab boundary could
  never land on a 256 cm segment boundary — every slab would straddle two segments' curvature by a
  different fraction. This was not a preference; the old value is arithmetically incompatible.
- `MC_TRI_MAX` 128 → **256**, and `mc_vtx_a`/`mc_vtx_b` `[1024]` → `[2048]` **together**. The arched
  section emits 192 triangles a frame, which does not fit in 128. Raising the cap without growing
  the arrays writes past the end of one list into the next.
- New palette (`src/tex.cyr`): the cold drowned shaft of the design doc rather than warm rock, with
  **emissive cyan rails** as the only bright thing in the frame. With no lighting available in
  op `0x0F` — the record carries no vertex colour and no modulation field — the rails are what make
  a curve legible at 100 km/h.
- Vertex lists, texture, ring and input pools all carry a named guard symbol, and every span is
  asserted as a `>=` byte comparison at startup.

### Fixed

- **`mc_emit`'s over-budget drop was silent and uncounted.** `if (n >= MC_TRI_MAX) { return 0; }`
  touched none of the drop counters, so a frame that quietly lost its last N triangles reported a
  perfectly clean tally — "0 dropped" over a tunnel with the far end missing. Now counted as
  `mc_drop_full` and held at zero by the gate.
- **The far end of the shaft was a hole.** op `0x0F` writes opaque black into every lane no triangle
  covered and the game cannot change that, so beyond the far plane sat a hard-edged black octagon
  that read as a hole in the world rather than as darkness. A far-cap quad now closes it; ride
  coverage went from 98.0% to **100%** of sampled pixels.

### Notes

- Peak `|D|` fell from **260,066,240 to 66,091,200** — an 8× ABI margin became **32×**. The slab
  ladder puts less of the frame at the near plane, where `w` is at the ABI floor and `D` is most
  sensitive.
- Degenerate (`cross == 0`) drops are no longer always zero: a curving track legitimately produces
  about one per frame, where two edges a fraction of a pixel apart round to the same integer column.
  They cover no pixel in any rasteriser. **A gate asserting all drop counts are zero will now fail**,
  which is why the pixel-losing drops are counted and asserted separately from the degenerate ones.

## [0.1.0] — 2026-07-30

First cut. A ride through a curving mine tunnel, drawn entirely by the AGNOS kernel's
perspective-correct textured triangle op — the first application outside the kernel's own test tree
to use the 3D ops at all.

### Added

- **The ride.** Eight tunnel segments, floor / ceiling / two walls each, 66 triangles a frame at
  640×400, presented through the framebuffer. The track curves via an interpolated 64-entry sine
  table, and the cart follows the rail with a lag rather than sitting on the centre line.
- **The GPU path** (`src/gpu.cyr`). `#86` carveout slots, `#72` upload, a **single `#92` dispatch
  carrying two op `0x0F` records** (floor+ceiling+background, then walls), `#90` readback out of the
  blit back buffer, `#73` copy out, `#39` present. Every syscall of that sequence was already proven
  on archaemenid by the kernel's rung-18 instrument.
- **Two 128×128 procedural textures** (`src/tex.cyr`) — gravel with timber sleepers and steel rails
  for the floor, hewn rock with ore glints for the walls. Power-of-two on both axes is mandatory:
  the shader masks with `w-1`/`h-1` on every fetch, so a non-power-of-two texture reads the wrong
  texel rather than wrapping, and the kernel rejects it outright.
- **The host gate** (`--check`, `src/check.cyr`). Re-derives the kernel's whole rule set from the
  **packed 64-byte records** — coordinate window, sub-pixel ban, `w` band, frame area, and the
  denominator bound at the draw rect's four corners — over 300 scripted frames, needing no GPU.
  Verdict at 0.1.0: 66 of 66 triangles emitted on every ride frame, zero pixel-losing drops, peak
  `|D|` 260,066,240 against a permitted 2,147,483,647 (an 8× margin), worst ride frame filling
  15,660 of 16,000 coverage samples.
- **Two assertions in the gate that are not ABI rules**: a coverage sweep with the background quad
  excluded, and a mutation that sets one fraction bit in an accepted record and requires the
  re-parse to catch it and name it `GPO_E_SUBPIXEL`.
- **Runtime buffer-size measurement.** The allocation unit of a module-global `var X[N]` has been
  measured to differ between files in this language, so the vertex lists, textures and frame buffer
  are checked against their required byte counts at startup rather than assumed.

### Notes

- **The frustum uses the ABI's full 16:1 depth range** (`z ∈ [32, 512]`, `w = 8z`, so `w` spans
  exactly `[256, 4096]`). That ratio is a ceiling, not a preference: `w` is proportional to depth and
  bounded to `[256, 4096]`, so no choice of scale can exceed 16:1.
- **Steering is not wired.** 0.1.0 is the ride; input is the next cut.
- Validated as an agnos binary under mirshi (byte-identical gate output to the host build) and
  loaded + run in ring 3 on the real kernel under QEMU. The GPU path itself is iron-only — `#86`
  slots come from the GPU carveout, which QEMU has no GPU to provide.

[Unreleased]: https://github.com/MacCracken/cyrius-mine-cart/compare/0.2.2...HEAD
[0.2.2]: https://github.com/MacCracken/cyrius-mine-cart/compare/0.2.1...0.2.2
[0.2.1]: https://github.com/MacCracken/cyrius-mine-cart/compare/0.2.0...0.2.1
[0.2.0]: https://github.com/MacCracken/cyrius-mine-cart/compare/0.1.0...0.2.0
[0.1.0]: https://github.com/MacCracken/cyrius-mine-cart/releases/tag/0.1.0
