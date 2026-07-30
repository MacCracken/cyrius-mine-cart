# Changelog

All notable changes to cyrius-mine-cart are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/), and this project adheres to
[Semantic Versioning](https://semver.org/).

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
