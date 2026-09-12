# Changelog

All notable changes to cyrius-mine-cart are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/), and this project adheres to
[Semantic Versioning](https://semver.org/).

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

[0.1.0]: https://github.com/MacCracken/cyrius-mine-cart/releases/tag/0.1.0

## [Unreleased]

## [0.2.1] - 2026-09-11

### Changed

- **Toolchain `6.4.78` → `6.6.2`.** No source change; the value form needed none.
  Build, tests and every bench/fuzz/distlib target re-verified at the new pin.
