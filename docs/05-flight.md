# 05 — Flight

Flight quality is the top priority of the whole game. Formulas and default values: [11-math.md](11-math.md) §1–10.

## What makes flight feel good

1. **Physics with inertia.** Thrust pushes along the nose, and the velocity catches up with the nose late, so turns carve and drift. With the engine off the rocket follows a ballistic arc.
2. **Camera.** A spring follow, FOV that widens with speed, shake.
3. **Feedback.** Flame, jet wash, sound, a wobble on landing.
4. **A stable simulation rate.** A fixed 60 Hz step with render interpolation. Without it, everything stutters on 120/144 Hz screens no matter how well it is tuned.

**Frame order:** all of this runs in **one** callback, `PreRender` (or `BindToRenderStep` at `Enum.RenderPriority.Camera.Value + 1`):
```
read input → N fixed sim steps → interpolate → set Root.CFrame → camera → effects
```
Keep that order in every flight controller; splitting it across callbacks brings jitter back.

## Controls: the camera aims, the rocket chases

The player steers the camera directly; the rocket does not snap to the crosshair, its nose **chases** the aim point. That gives precise aiming and real inertia. The small "x" crosshair at the top of the reference screen works exactly this way.

- **Aim point:** a 500-stud ray from the camera through the crosshair; the nose targets the hit point ([11-math §2](11-math.md)). Without this parallax correction the rocket misses windows in tight interiors.
- **PC:** mouse locked to the centre. Thrust: LMB, Space or W. Detonate: E.
- **Gamepad:** right stick aims. RT is **analog throttle** (`trigger^1.5`). Detonate: X.
- **Mobile:**
  - the left half of the screen is an aim touchpad (relative drag, response curve `|Δ|^1.3`);
  - a big thrust button bottom-right, a smaller detonate button above it;
  - only that panel stays visible during flight.
- Everything goes through `Branch.Keybinds`: contexts `Lobby` and `Flight`. Mobile buttons bind through `InputBinding.UIButton`; analog throttle is a `Direction1D` action.
- **Input buffer:** thrust pressed during the landing settle starts a takeoff immediately.

## State machine

| State | Enter when | Behaviour |
|---|---|---|
| `Standing` | Run start (on the pad), or landing settled | Upright, no simulation except the support check. Thrust ⇒ `Flying`. After 0.3 s still ⇒ **disguised**. On PRO, fuel refills. |
| `Flying` | Throttle > 0.05 | Full force model ([§4–6](11-math.md)) |
| `Coasting` | Airborne, throttle ≤ 0.05 | Same model; weathervane active, low steering authority |
| `Landing` | Floor contact that passes the landing test ([§7.5](11-math.md)) | Settle spring with low damping, pivoting on the base ([§8](11-math.md)) ⇒ `Standing` |
| `Falling` | Standing and the support disappears | Same as `Coasting` |
| `Exploded` | Crash, detonation, or the server ends the run | Simulation stops; kill cam |

## Flight model in one table

| Piece | What it does for the feel | Math |
|---|---|---|
| Engine spool + ignition kick | Punch without lag | §3.1–3.2 |
| Nose spring, authority 0.35 + 0.65·throttle | Sharp with the engine on, floaty when coasting | §4.2–4.3 |
| Weathervane | The nose follows the arc with the engine off | §4.4 |
| Turn-rate cap that drops with speed | Mass: fast means wide turns | §4.5 |
| Own gravity, 60 | Readable hops and arcs | §5.1 |
| Thrust taper | Soft top speed | §5.2 |
| Low drag | Momentum is kept: glides | §5.3 |
| Fin lift ∝ speed² | Fast flies straight, slow sags | §5.4 |
| Ground cushion | Soft landings, easy low hover indoors | §5.5 |
| Grip (rotate + bleed) | Carve vs drift: the main "character" knob | §6 |
| Hard speed cap | Fair, server-checkable limit | §6.3 |
| Visual bank and vibration | Motion read without touching physics | §9 |

## Surfaces

- **Sweep:** two spherecasts, nose and centre, radius 0.45, continuous: no tunnelling ([§7.1](11-math.md)).
- **Crash:** only a hard, nose-first hit (`noseIn > 0.82`, `vn > 42`) explodes. A crash next to a target counts as a strike ([07](07-guards-targets.md)).
- **Slide:** everything else slides, with impact friction, a small bounce above 8 studs/s, and time-based friction in sustained contact ([§7.6](11-math.md)). Glancing hits throw cube sparks.
- **Landing:**
  - the rocket stands up with an under-damped spring, rocking two or three times like a bottle;
  - it **pivots on its base**, not its centre;
  - cube dust and a "thunk".
- **Standing:** a support ray every step. If the support disappears (a pushed crate, an edge), the rocket falls.
- **Takeoff:**
  - the first 0.15 s blend thrust toward world up, so aiming at the floor does not plough into it;
  - crashes are ignored during those 0.15 s.

## Camera

| Aspect | Behaviour | Math |
|---|---|---|
| Orientation | Player-controlled yaw/pitch; touch response curve | §10.1 |
| Position | Behind and above, `(0, 2.2, 8.5)`; drops back up to 2 studs under acceleration | §10.2 |
| Follow | Critically damped spring, smooth time 0.08 s | §10.3 |
| Walls | Pull in instantly, ease out over 0.25 s | §10.4 |
| FOV | 70 → 82 with speed, +4 kick on ignition | §10.5 |
| Shake | Trauma model: engine, jet wash, impacts, explosions | §10.6 |
| Standing | Closer (6.5), free orbit around the rocket | — |
| Kill cam | Time ×0.3 for 0.6 s (own sim + `ParticleEmitter.TimeScale`), orbit around the blast | — |

During flight, `Camera.CameraType = Scriptable`.

## Tuning

Every constant in the flight model is an attribute on `ReplicatedStorage.Shared.FlightTuning` (a Configuration instance), named after the parameters in [11-math.md](11-math.md): `Gravity`, `ThrustAccel`, `MaxSpeedBase`, `HardCapMul`, `DragLinear`, `DragQuad`, `LiftCoef`, `GripBase`, `GripPerSpeed`, `SlideBleed`, `NoseFrequency`, `NoseDamping`, `TurnRateSlow`, `TurnRateFast`, `CoastAuthority`, `WeathervaneFreq`, `ThrottleUpTime`, `ThrottleDownTime`, `IgnitionKick`, `CushionStrength`, `CrashSpeed`, `CrashNoseCos`, `LandSpeed`, `ImpactFriction`, `SlideFriction`, `Bounce`, `SettleFrequency`, `SettleDamping`, `BankMax`, `BankGain`, `CamDistance`, `CamHeight`, `CamSmoothTime`, `FovBase`, `FovSpeedBonus`.

- `RocketController` reads them every frame, so values can be changed in Studio's Properties panel **during Play**. Values that feel right get copied back into the file's defaults.
- The server reads only what it validates against (`MaxSpeedBase`, `HardCapMul`).

**Debug view** (a toggle, Studio only):
- velocity vector, nose vector, aim point;
- spherecast hits with their normals;
- jet wash ray and intensity;
- a text overlay with state, speed, throttle and fuel.

**Flight gym:** a dev-only course in a test place:
- rings at different heights;
- a 3×2-stud window and a corridor with a 90° bend;
- a table, stairs, a low ceiling;
- a long straight for top-speed checks.

**Target metrics** (at `vmax = 60`)

| Metric | Target |
|---|---|
| 0 → 90% of top speed | ≈ 0.6 s |
| 180° reversal at speed | ≈ 0.6 s |
| Window 3×2 from 30 studs, first try (new player) | most attempts |
| Landing on a table without crashing | within 2–3 tries |
| Hover just above a floor using the cushion | possible after a minute of practice |

Playtest on PC, phone and gamepad before moving to networking ([12-roadmap.md](12-roadmap.md), phase 2d).

## Network implications

- The client sends `{ t, cf, vel, throttle }` at 30 Hz ([03](03-network.md)).
- The server checks the displacement against the hard cap, compares the reported velocity with the actual displacement, ray-checks walls, and runs fuel on PRO ([11-math §15](11-math.md)).
- Other players' rockets use cubic Hermite interpolation with velocity, 100 ms in the past ([§15.7](11-math.md)). Their flame and jet wash are driven by the replicated `throttle`.
- A `Correct` from the server snaps the local rocket back to the last valid state and keeps its velocity. It should be rare; log it in Studio.
