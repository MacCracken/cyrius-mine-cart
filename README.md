# cyrius-mine-cart

A ride through a curving mine tunnel, textured and perspective-correct, rasterised by the AGNOS
kernel's GPU.

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
| *(default)* | Ride the tunnel. Needs AGNOS and a GPU. |
| `--check` | The host gate — build frames and re-validate every record the kernel would see. Runs anywhere, needs no GPU. Exit 95 = pass. |
| `--frames N` | Bound the ride (0 = unbounded). |
| `--trace` | Report every dropped triangle, for diagnosis. |

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

640×400, two 128×128 procedural textures, eight tunnel segments, 66 triangles a frame. Segments are
drawn far-to-near: op `0x0F` replaces the destination pixel rather than compositing, and a tunnel is
convex from the inside, so painter's order *is* the depth test and no z-buffer is needed.

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

`0.1.0` — the ride. The cart follows the rail with a lag rather than sitting on the centre line, so
the tunnel sways past. Steering input is not wired yet; that is the next cut.

## License

GPL-3.0-only.
