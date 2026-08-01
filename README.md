# cyrius-mine-cart — DEEPVEIN

Shaft 9 runs under a drowned arcology. You take the core, ride the rails down, and outrun what
follows. Textured, perspective-correct, rasterised by the AGNOS kernel's GPU.

This is the **first application outside the kernel's own test tree to draw with the AGNOS 3D ops**.
The kernel grew triangle rasterisation on real AMD gfx90c silicon over a long ladder of rungs —
edge coverage, barycentric colour, triangle lists, affine texturing, batched primitives, a depth
test, and finally perspective-correct interpolation — and every one of those rungs closed on a
proof binary. Proof binaries are the right way to build a rasteriser and the wrong way to know you
have one. This is the thing that was being built toward: a camera, a texture, a frame, on screen.

Written in [Cyrius](https://github.com/MacCracken/cyrius). No libc, no floating point, no external
dependencies. Every pixel goes through `#92 gpu_shader_op` op `0x0F`.

## Running it

```bash
cyrius build src/main.cyr build/mine-cart
./build/mine-cart --check
```

| Mode | What it does |
|---|---|
| *(default)* | Ride. Needs AGNOS and a GPU. A/D or arrows lean, SHIFT brakes, ESC quits. |
| `--check` | The host gate — 164 self-test assertions, then build frames and re-validate every record the kernel would see. Runs anywhere, needs no GPU. Exit 95 = pass. |
| `--sim` | Run the simulation headless and print telemetry. The tuning instrument. |
| `--pilot N` | With `--sim`: 0 = hands off the controls, 1 = crude autopilot. |
| `--verify` | One frame, GPU vs CPU reference, byte-compared. |
| `--frames N` | Bound the ride (0 = unbounded). |
| `--trace` | Report every dropped triangle, for diagnosis. |

## The world is the track

There is no 3D scene. A flat ring of 256 cm segments, each carrying a curvature and a pitch, **is**
the entire world — a few kilobytes of integers. The renderer walks it far-to-near and turns
per-segment curvature into a centre line by exact running integral; the simulation walks it as a
`pos_seg` + `pos_sub` position with a real carry. 96 segments live in a 128-slot ring, 48 generated
ahead, discarded behind.

The ring is 128 slots so the index is a **mask**, not a modulo. `pos_seg % 96` works perfectly for
two hours and twenty minutes and then, at the u16 wrap, jumps the ring 64 entries sideways in a
single tick. The authoritative position is an absolute monotonic counter; the u16 the design calls
for is a *view* of it.

## The loop

The cart always moves forward. You control speed and lean, nothing else.

Lateral force is **`curve × speed²`**. Squaring the speed is what makes the brake a decision rather
than a penalty — at half speed a bend pushes a quarter as hard, so braking into a curve buys far
more control than the time it costs. Braking in and releasing at the apex pays a speed bonus;
sitting on the brake through the whole bend pays nothing, because otherwise the safest line is also
the slowest and the game becomes a patience test.

**96 km/h is the throttle ceiling. 103 km/h has to be earned.**

Lean past the threshold for 30 consecutive ticks derails. There is no lean bar: lean *is* the
camera's lateral offset, so drifting wide brings the wall closer. The margin is felt, not read.

## Determinism

Fixed 60 Hz simulation, integer RNG seeded once, **zero wall-clock reads inside the sim**. The
simulation sees exactly one input word per tick — not a keyboard, not a scancode, not a device — so
same seed plus same input log reproduces the run exactly. Rendering may drop frames; the simulation
never does. On a target with no debugger that replay is the only regression test that exists, which
is why the boundary is drawn this narrowly.

On AGNOS it is staged as `/bin/mine-cart` by the kernel repo's `scripts/burn/stage-tools.sh`.

## How a frame is drawn

```
  #86 shm_alloc      once — two vertex slots, two texture slots, one readback slot
  #72 shm_write      upload this frame's vertex lists (textures go up once, at init)
  #92 gpu_shader_op  TWO records — floor + ceiling + background, then walls. ONE dispatch.
  #90 gpu_readback   copy the drawn rect out of the blit back buffer
  #73 shm_read       carry it into the game's own buffer
  #39 blit           present
```

640×400, one 128×128 procedural texture, sixteen slabs off an 8/16/32/64 render-unit ladder, **192
triangles a frame** in a single record. Near slabs are short and far ones long: a uniform walk makes
the nearest slab 280 px tall on a 400 px screen and the farthest 2 px, so a curve reads as a polygon
rather than a bend.

Slabs are drawn far-to-near: op `0x0F` replaces the destination pixel rather than compositing, and a
tunnel is convex from the inside, so painter's order *is* the depth test and no z-buffer is needed.

**There is no lighting.** The record carries no vertex colour and no modulation field, so distance
fog and lamp falloff are not available at all. The palette stays dark and cold and the emissive cyan
rails carry every bit of the contrast — they are what make a bend legible at 100 km/h.

## Why `--check` exists

`syscall(92, ...)` validates **every** record before dispatching **any** of them. One vertex with a
fraction bit set does not draw one wrong triangle — it rejects the whole batch, and the frame comes
back as whatever was in the slot before. On the target hardware that is a black screen, and a black
screen is indistinguishable from "no GPU", "shader wrote nothing", and "readback failed".

So `--check` re-derives the kernel's entire rule set — coordinate window, sub-pixel ban, the `w`
band, frame area, the denominator bound at the draw rect's four corners — **from the packed 64-byte
records**, never from the values that were passed in. Checking the emitter's inputs would only prove
the inputs were sane; this checks the bytes the kernel will actually read.

It also asserts two things that are not ABI rules at all:

- **Coverage.** A frame can satisfy every rule and still project to nothing. Legal-and-empty is the
  one failure a burn cannot tell apart from a refused batch, so the gate measures how much of the
  screen the tunnel fills — with the full-screen background quad *excluded*, because counting it
  would answer "yes" for a frame in which every tunnel triangle had collapsed.
- **That the gate can fail.** It mutates an already-accepted record — one fraction bit, the exact
  defect that would cost a flash — and requires the re-parse to catch it and name it correctly.

## Geometry notes

**The `w` band is a 16:1 depth ceiling.** `w` must lie in `[256, 4096]` and is proportional to
depth, so `z_far / z_near` can never exceed 16 no matter what scale is chosen — a larger scale pulls
the near plane closer *and* the far plane in by the same factor. The frustum uses exactly that range
(`z ∈ [32, 512]`, `w = 8z`). An earlier 8:1 setting left half the depth unused, and the visible cost
was that geometry nearer than the near plane is not drawn — precisely what should sweep out to the
sides and fill the screen when the cart passes close to a wall.

**No near-plane clipper.** The segment grid is anchored *to* the near plane, so nothing is ever
behind the camera. That is deliberate: a clipper is where sub-pixel coordinates and degenerate
triangles come from, and both of those are whole-batch rejections.

**Degenerate triangles are dropped, and that is not a hole.** The kernel's area floor is `2^16` and
the area is `|cross| << 16`, so the rule can only fire when `cross` is *exactly* zero — the triangle
is precisely collinear and covers no pixel in any rasteriser. Distant tunnel segments produce these
routinely: two wall edges a fraction of a pixel apart round to the same integer column, and the ABI
forbids sub-pixel coordinates. A `w`-band or coordinate-window drop is a different matter and the
gate holds those to zero.

## Why it is fullscreen and not a window

Not because there is no desktop — there is. aethersafha runs on AGNOS, grants windows, and already
composites client surfaces on the GPU; setu even asks the kernel for a client's buffer as a `#86`
carveout slot, so the client→compositor transport is already GPU-visible memory.

The boundary is that **no GPU op that writes colour lets the caller name where the colour goes.**
`gpu_tri_persp` derives its destination from two kernel module globals (`gpu_bb_a_mc`/`gpu_bb_b_mc`),
and the `#92` record has no field to override it — the op's field mask is `0x00FF`, dwords 0–7, and the
validator rejects anything outside it. So a GPU draw lands in the *same* shared back buffer the
compositor stages into between its deferred flip stage and its flip, and `#92` carries no process
identity to arbitrate. A GPU-rendering client today can own the whole frame or corrupt someone else's.

The fix already exists for depth: `gpu_tri_depth` writes Z to a ring-3-named handle while writing colour
to the back buffer, in the same dispatch. Colour never got the field depth has.

## Status

`0.2.0` — **the loop, and deliberately nothing more.** The build order this game is being written
against says to stop after speed/brake/lean/derail and re-tune if it is not fun with nothing else in
it, because nothing added later will save it. So that is where this stops.

Measured with `--sim` over 30 s: hands off the controls the cart derails about every 7 s and reaches
203 m; a crude autopilot reaches 787 m with 7 clean apexes and no derails. That gap is the
difficulty, and it is the number to argue with.

Not yet built, in the order they come: duck/jump obstacles, forks and their 1.2 s telegraph, pickups
and score, pursuers and combat, the flood finale, synthesised audio, and the input-log replay test.
The input word boundary and the per-chapter seeding are already in place for that last one.

## License

GPL-3.0-only.
