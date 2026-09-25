# DEEPVEIN roadmap

The game is built against a design brief with an explicit build order, and against the visual design
doc in [`Deepvein.dc.html`](../../Deepvein.dc.html). This file is where that order stands. Volatile
numbers (triangle counts, `|D|` margins, sim telemetry) live in the README and the CHANGELOG, not here.

## The build order

| Step | What | State |
|---|---|---|
| 1–3 | Fixed-point core, the segment ring and its streaming generator, the ring-driven renderer | done — 0.2.0 |
| 4 | **The loop**: speed, brake, lean, derail — "fun with nothing else in it" | done — 0.2.0; grip/slip model 0.2.1 |
| 5 | Duck and jump obstacles: beams, gaps, integrity | done — 0.2.1; drawn where they resolve, 0.2.2 |
| 6 | Forks, and their 1.2 s telegraph | done — unreleased: the switch, the sign (plate 1A), the wet shaft and the spur; follow-ups below |
| 7 | Pickups and score | done — 0.2.1; visible for the first time, 0.2.2 |
| 8 | Pursuers and combat (the whip) | **not started** — scaffolding only |
| 9 | The flood finale | **not started** — scaffolding only |
| 10 | Synthesised audio | done — 0.2.1 (`--wav` to listen) |
| 11 | The input-log replay test | done — 0.2.2 (`--record`, `--replay`, `film --log`, and in `--check`) |

## Next, and what each one needs

### Available now (no external blocker)

- **Run it on the hardware.** Nothing since 0.2.1 has been on a GPU — neither 0.2.2's changes
  (culling, the camera, the atlas, the HUD) nor the forks. In order: `run /bin/mine-cart --check`
  (must print exit 95 on agnos too), `--verify` (frame 120, GPU vs CPU reference, must be
  `DIFFER: 0`), `--verify --pilot 1 --frames 715` (a frame inside a fork — hands off, frame 120 never
  reaches one), then a ride with `--record /tmp/ride.dvrp`, and `--replay` of that log on a host.
  `--verify` is the only instrument that can see a picture that is ABI-legal and wrong.
  - agnos's `scripts/burn/stage-tools.sh` copies `build/mine-cart_agnos` from this repo as committed.
  - A log records which simulation made it (0.2.2 is revision 0, forks revision 1), and a build
    refuses another revision's log by name — replay a log with the build that recorded it.
- **Tuning — step 4's question, and now step 6's.** `--sim` says: hands off, the cart falls into the
  first gap at 87 m every time and never reaches a fork; the reference autopilot takes every fork's wet
  shaft and still never dies in two minutes, though it slips measurably more there (the CHANGELOG has
  the numbers). Whether the wet shaft's slick rails (0.80 of dry grip) and closer hazards are worth ×2
  is a question for a person on iron — the autopilot reads the ring and has no reaction time, so it is
  a floor on the difficulty, not a verdict. The spur's hazard spacing is the trunk's for now; its own
  knobs are in `config.cyr`.
- **Run summary / death screens** (design plates 1D, 1E). A death restarts the chapter instantly
  today, with the cause printed to the console and nothing on screen — and the HUD shows no score, so
  the wet shaft's ×2 is never seen during play. The summary is where the fork's reward becomes
  visible. Needs letters in the atlas: the HUD font is digits only, and the fork sign's glyphs (W, E,
  T, D, R, Y) are the start of one.
- **Rail hops on the three-track runs.** The input word already carries `IN_HOP_LEFT/RIGHT` (jump + a
  direction) and the tracks are drawn; the simulation ignores both. Open design questions, in the
  order they bite: what the self-centring spring pulls toward while on an outer rail (today, the
  tunnel centre); what happens to a cart on an outer rail when the run ends (the side tracks need to
  merge, visibly, or end in a buffer); and whether the derail threshold stays measured from the
  tunnel centre (config.cyr says it does, which spends 61% of the budget just riding an outer rail).
  A fork cuts a three-track run short at its sterile approach, so a hop cannot carry into one.
- **Distance fog.** op `0x0F` has no lighting and no vertex colour, so the far tunnel is as bright as
  the near one. A second, *compositing* op in the same `#92` dispatch can darken it: op `0x0A`
  (`GPU_OP_TRI_LIST`) interpolates premultiplied RGBA per vertex and composites src-over, and the
  op-to-op prep-arena race was fixed in agnos 1.56.33 — the kernel side exists. Buildable on the host:
  a 48-byte-vertex emitter for `0x0A`, `refcore` modelling src-over so `--verify` can compare it, and
  the gate's "exactly one record" narrowed to "exactly one `0x0F` record". ⚠ `0x0A` does not bin: its
  budget is `w × h × triangles × 3 ≤ 2^26` over its own rect (agnos `kernel/core/gpu.cyr:105`), at
  most 87 triangles at 640×400, each ~0.46 ms by the kernel's own fit if it spans the rect. So fog is
  a few bands in a tight rect around the far bore, and its real cost is an iron measurement. Better
  done after the burns above, so a picture that goes wrong has one new op to blame, not two.

### Fork follow-ups (step 6, as shipped)

- **Reconvergence is not drawn.** A branch runs 156–312 segments, lays its last `CFG_BRANCH_TAIL` as
  trunk again, and then simply is the trunk. The other branch rejoining — a side junction — is not
  drawn. It would be the fork's mirror: a dark mouth in one wall.
- **The unlit third branch** (`FLAG_SHORTCUT`, "an unlit third branch is the shortcut") is not built:
  forks are two-way.
- **The plate's "FORK IN 140 m"** is not shown. The 1A sign is world-space, so it is visible from the
  far plane, 1.26 s out at top speed; a distance read-out further out would be a HUD element, which
  1A deliberately is not.
- **Wet stays left.** The plates put WET SHAFT on the left, and so does every fork; which side is wet
  could be drawn per fork, if knowing the answer in advance turns out to be a problem.

### Waiting on something else

- **Pursuers and the whip (step 8), the flood (step 9).** Audio scaffolds exist (`aud_trigger_crack`,
  `aud_set_flood`) and `IN_WHIP` is decoded. Both are content work that should follow a **played**
  step 6, per the brief's order — they wait on a person on iron, not on the kernel.
  - ⚠ **A run never ends.** `cart_chapter` is never incremented, so the game is one endless chapter
    and every death restarts at 0 m; the design's 60–90 s descent does not exist yet. The flood is the
    chapter's finale, so chapter progression belongs with step 9.
- **A window instead of fullscreen.** The transport question is settled — setu 0.8.0 speaks the
  agnos channel band (`#97 chan_op`), and puka has presented a composited window since 2026-08-07. The
  one blocker left is that no colour-writing GPU op lets the caller name its destination (see the
  README); agnos lists that as an open bite, not scheduled.

## Instruments, for whoever picks this up

| Question | Instrument |
|---|---|
| Will the kernel accept it? | `mine-cart --check` / `cyrius tests` |
| What does it look like? | `tools/shot.cyr` (stills; `--pilot 2` takes the spur), `tools/film.cyr` (video) |
| Does the GPU draw what the reference does? | `mine-cart --verify` on iron — `--pilot 1 --frames N` for a frame of the autopilot's ride |
| Is it the same ride? | `--record` on the ride, `--replay` on a host, `film --log` to watch it |
| Does it sound right? | `mine-cart --wav`, and a person |
| Is it fun? | `--sim` for the numbers, and a person on iron for the answer |
