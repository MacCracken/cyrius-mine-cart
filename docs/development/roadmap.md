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
| 6 | Forks, and their 1.2 s telegraph | **not started** — scaffolding only (below) |
| 7 | Pickups and score | done — 0.2.1; visible for the first time, 0.2.2 |
| 8 | Pursuers and combat (the whip) | **not started** — scaffolding only |
| 9 | The flood finale | **not started** — scaffolding only |
| 10 | Synthesised audio | done — 0.2.1 (`--wav` to listen) |
| 11 | The input-log replay test | done — 0.2.2 (`--record`, `--replay`, `film --log`, and in `--check`) |

## Next, and what each one needs

### Available now (no external blocker)

- **Burn 0.2.2 on iron.** Nothing in this release has been on a GPU. In order: `run /bin/mine-cart
  --check` (must print exit 95 on agnos too), `--verify` (frame 120, GPU vs CPU reference, must be
  `DIFFER: 0`), then a ride with `--record /tmp/ride.dvrp`, and `--replay` of that log on a host. The
  visual changes are large — culling, the camera placement, the atlas, the HUD — and `--verify` is the
  only instrument that can see a picture that is ABI-legal and wrong.
- **Rail hops on the three-track runs.** The input word already carries `IN_HOP_LEFT/RIGHT` (jump + a
  direction) and the tracks are drawn; the simulation ignores both. Open design questions, in the
  order they bite: what the self-centring spring pulls toward while on an outer rail (today, the
  tunnel centre); what happens to a cart on an outer rail when the run ends (the side tracks need to
  merge, visibly, or end in a buffer); and whether the derail threshold stays measured from the
  tunnel centre (config.cyr says it does, which spends 61% of the budget just riding an outer rail).
- **Forks (step 6).** In the ring already: `KIND_FORK`, `FLAG_LIT` / `FLAG_SHORTCUT`, the branch
  fields (`trk_branch_a/b`), `CFG_BRANCH_*` reconvergence lengths, and a double-tap SWITCH in the
  input word (`IN_SW_LEFT/RIGHT`) designed so leaning and switching can share keys. Missing: the
  generator laying a fork and two diverging branches, the renderer drawing a split bore (the one
  record has room: the busiest frame is 231 of 256 and 32 are held in reserve), the telegraph, and
  the design plates' risk/yield choice ("WET SHAFT ×2 / SERVICE SPUR ×1").
- **Tuning — step 4's question, asked again.** `--sim` says: hands off, the cart falls into the first
  gap at 87 m every time; the reference autopilot never dies in two minutes. The gap between them is
  the difficulty, and nobody has played it yet to say whether it is right.
- **Run summary / death screens** (design plates 1D, 1E). A death restarts the chapter instantly
  today, with the cause printed to the console and nothing on screen.

### Needs the kernel, or iron, first

- **Distance fog.** op `0x0F` has no lighting and no vertex colour, so the far tunnel is as bright as
  the near one. A second, *compositing* op in the same `#92` dispatch can darken it: op `0x0A`
  (`GPU_OP_TRI_LIST`) interpolates premultiplied RGBA per vertex and composites src-over, and the
  op-to-op prep-arena race that once made two records in one dispatch draw the second list twice
  was fixed in agnos 1.56.33. It needs: a 48-byte-vertex emitter for `0x0A` (a different record from
  `0x0F`'s 64 bytes), `refcore` modelling src-over so `--verify` can compare it, the gate's
  "exactly one record" assertion narrowed to "exactly one `0x0F` record", and an iron measurement of
  the cost — the kernel calls its fix a stopgap that suspends batching around each dispatch.
- **Pursuers and the whip (step 8), the flood (step 9).** Audio scaffolds exist (`aud_trigger_crack`,
  `aud_set_flood`) and `IN_WHIP` is decoded. Both are content work that should follow a played
  step 6, per the brief's order.
- **A window instead of fullscreen.** Blocked on the kernel: no colour-writing GPU op lets the caller
  name its destination (see the README), and the setu transport is moving to the agnos socket.

## Instruments, for whoever picks this up

| Question | Instrument |
|---|---|
| Will the kernel accept it? | `mine-cart --check` / `cyrius tests` |
| What does it look like? | `tools/shot.cyr` (stills), `tools/film.cyr` (video) |
| Is it the same ride? | `--record` on the ride, `--replay` on a host, `film --log` to watch it |
| Does it sound right? | `mine-cart --wav`, and a person |
| Is it fun? | `--sim` for the numbers, and a person on iron for the answer |
