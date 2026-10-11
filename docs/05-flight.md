# 05 — Flight

Flight quality is the top priority of the whole game. Formulas and default values: [11-math.md](11-math.md) §1–10. Calibration against the reference video: [13-reference-physics.md](13-reference-physics.md).

## The model in one sentence

The player rotates the rocket at any time with WASD, Space pushes it along its nose, and the velocity stays the velocity until thrust, gravity, drag or a surface changes it. **Inertia is the core.** The rocket can point one way and keep sliding another; to stop, you turn it against your motion and burn.

## What makes flight feel good

1. **Pure inertia with punchy thrust.** Momentum carries. Short taps make distinct hops; long burns build speed. Nothing steers the velocity for you.
2. **Rotation that is always available,** with a little angular inertia: taps nudge, held keys spin fast, release holds the attitude.
3. **Camera.** A free orbit that follows the rocket almost rigidly, with the rocket in the lower third of the screen.
4. **Feedback.** Flame, nozzle ring, engine light on nearby surfaces, jet wash, sound, a clean self-righting on landing.
5. **A stable simulation rate.** A fixed 60 Hz step with render interpolation. Without it, everything stutters on 120/144 Hz screens no matter how well it is tuned.

**Frame order:** all of this runs in **one** callback, `PreRender` (or `BindToRenderStep` at `Enum.RenderPriority.Camera.Value + 1`):
```
read input → N fixed sim steps → interpolate → set Root.CFrame → camera → effects
```
Keep that order in every flight controller; splitting it across callbacks brings jitter back.

## Controls

| Action | PC | Gamepad | Mobile |
|---|---|---|---|
| Rotate the rocket | W/S tip the nose away from / toward the camera; A/D move the nose left / right on screen | Left stick | Virtual stick, left side |
| Thrust | Space (hold or tap) | RT, analog (`trigger^1.5`) | Big thrust button, bottom right |
| Detonate | E | X | Small button above thrust |
| Camera orbit | Right mouse drag (Roblox-style); Shift toggles mouse lock | Right stick | Drag on the right half of the screen |
| Zoom | Mouse wheel | D-pad up/down | Pinch |

- Rotation axes come from the camera's **yaw**, so "W" always means "away from me" ([11-math §2](11-math.md)):
  - upright, W/S/A/D lean the rocket forward, back, left and right;
  - flying level, W/S dive and climb, A/D turn.
- Everything goes through `Branch.Keybinds`: contexts `Lobby` and `Flight`. The stick is a `Direction2D` action, analog thrust a `Direction1D` action; mobile buttons bind through `InputBinding.UIButton`.
- **Input buffer:** thrust pressed during the landing settle starts a takeoff immediately.

## State machine

| State | Enter when | Behaviour |
|---|---|---|
| `Standing` | Run start (on the pad), or landing settled | Upright on the fins. WASD tilts it up to 35° on its base to aim the takeoff; with no input it springs back upright. Thrust ⇒ `Flying`. After 0.3 s still and upright ⇒ **disguised**. On PRO, fuel refills. |
| `Flying` | Throttle > 0.05 | Gravity + thrust along the nose + drag + cushion ([§5](11-math.md)) |
| `Coasting` | Airborne, throttle ≤ 0.05 | Same forces without thrust; full rotation still available |
| `Landing` | Floor contact that passes the landing test ([§7.5](11-math.md)) | Fast, clean self-righting spring pivoting on the base ([§8](11-math.md)) ⇒ `Standing` |
| `Falling` | Standing and the support disappears | Same as `Coasting` |
| `Exploded` | Crash, detonation, or the server ends the run | Simulation stops; kill cam |

## Flight model in one table

| Piece | What it does for the feel | Math |
|---|---|---|
| Camera-relative rate control with angular inertia | Taps nudge, held keys flip fast, release holds the attitude | §2, §4 |
| Engine spool 0.05 / 0.08 s + ignition kick | Every tap registers as a distinct push | §3.1–3.2 |
| Gravity 110, thrust 360 (T/g ≈ 3.3) | Snappy hops: a 0.13 s tap lifts about one rocket length | §5.1–5.2 |
| Thrust taper along the nose only | Soft top speed; braking against your motion is always full power | §5.2 |
| Low drag | Momentum carries; you plan your stops | §5.3 |
| Ground cushion (small) | Softer landings, easier low hovering | §5.4 |
| Speed cap `1.5·vmax + 20` | Fair, server-checkable limit with room for falls | §6 |

What is deliberately **not** in the model: velocity alignment ("grip"), fin lift, weathervaning, steering authority tied to throttle, automatic levelling in the air. The reference has none of them ([13 §10](13-reference-physics.md)). Optional assists for younger players are in the [backlog](backlog.md).

## Surfaces

- **Sweep:** two spherecasts, nose and centre, radius 0.45, continuous: no tunnelling ([§7.1](11-math.md)).
- **Crash:** only a hard, nose-first hit (`noseIn > 0.82`, `vn > 45`) explodes. A crash next to a target counts as a strike ([07](07-guards-targets.md)).
- **Slide:** everything else slides, with impact friction, a small bounce above 8 studs/s, and time-based friction in sustained contact ([§7.6](11-math.md)). A tilted rocket skimming the floor slides on its fins between taps, as in the reference. Glancing hits throw cube sparks.
- **Landing:** floor contact with speed < 25, no thrust and tilt < 60°.
  - The rocket rights itself with a near-critically-damped spring (25° → upright in about 0.15–0.3 s, no visible wobble), **pivoting on its base**.
  - Cube dust and a "thunk".
- **Standing:** a support ray every step. If the support disappears (a pushed crate, an edge), the rocket falls.
- **Takeoff:**
  - leaves the ground along the (WASD-tilted) nose;
  - the first 0.15 s blend thrust toward world up, so a takeoff tilted at the floor does not plough into it;
  - crashes are ignored during those 0.15 s.

## Camera

| Aspect | Behaviour | Math |
|---|---|---|
| Orbit | Player-controlled yaw/pitch, independent of the rocket's attitude | §10.1 |
| Framing | Looks at a point 1.2 rocket lengths above the rocket: it sits in the lower third | §10.2 |
| Distance | 12 studs by default, zoom 8–25 | §10.2 |
| Follow | Near-rigid critically damped spring, smooth time 0.04 s | §10.3 |
| Walls | Pull in instantly, ease out over 0.25 s | §10.4 |
| FOV | 70 → 80 with speed, +3 kick on ignition (our addition; subtle) | §10.5 |
| Shake | Trauma model: engine, jet wash, impacts, explosions | §10.6 |
| Kill cam | Time ×0.3 for 0.6 s (own sim + `ParticleEmitter.TimeScale`), orbit around the blast | — |

During flight, `Camera.CameraType = Scriptable`; the orbit is implemented by `CameraController`, not Roblox's default camera, so it obeys the frame order above.

## Tuning

Every constant in the flight model is an attribute on `ReplicatedStorage.Shared.FlightTuning` (a Configuration instance), named after the parameters in [11-math.md](11-math.md):

`Gravity`, `ThrustAccel`, `MaxSpeedBase`, `MaxSpeedPerLevel`, `SpeedCapMul`, `SpeedCapAdd`, `DragLinear`, `DragQuad`, `CushionStrength`, `SpinMax`, `SpinAccel`, `SpinBrake`, `StandTiltMax`, `ThrottleUpTime`, `ThrottleDownTime`, `IgnitionKick`, `CrashSpeed`, `CrashNoseCos`, `LandSpeed`, `LandTiltMax`, `ImpactFriction`, `SlideFriction`, `Bounce`, `SettleFrequency`, `SettleDamping`, `CamDistance`, `CamFocusHeight`, `CamSmoothTime`, `FovBase`, `FovSpeedBonus`.

- `RocketController` reads them every frame, so values can be changed in Studio's Properties panel **during Play**. Values that feel right get copied back into the file's defaults.
- The server reads only what it validates against (`MaxSpeedBase`, `MaxSpeedPerLevel`, `SpeedCapMul`, `SpeedCapAdd`).
- **Gravity presets** to compare first: 80 / 110 / 150 / 196, with `ThrustAccel = 3.3 · Gravity` ([13 §5](13-reference-physics.md#gravity)).

**Debug view** (a toggle, Studio only):
- velocity vector and nose vector, drawn separately so the inertia is visible;
- spherecast hits with their normals;
- jet wash ray and intensity;
- a text overlay with state, speed, throttle, angular speed and fuel.

**Flight gym:** a dev-only course in a test place:
- rings at different heights;
- a 3×2-stud window and a corridor with a 90° bend;
- a table, stairs, a low ceiling;
- a long straight for top-speed checks;
- a "stop on the mark" pad 60 studs away (tests braking by flipping).

**Target metrics**, matched to the reference ([13](13-reference-physics.md))

| Metric | Target |
|---|---|
| Vertical 0.13 s tap from standing | Rises about 1–1.3 rocket lengths (simulated: 6.2 studs, 0.8 s airtime to the same height) |
| Tap at 45° tilt | Airtime about 0.5 s, about 14 studs forward (simulated); a 0.6 s tap rhythm moves you along the floor |
| Hover upright | Possible with about 31% throttle or steady tapping |
| Hold-key 180° flip | ≈ 0.4 s |
| 0.05 s key tap | Nose moves about 10° |
| Cruise speed outdoors (base upgrades) | ≈ 35–40 studs/s |
| Stop from cruise by flipping and burning | Within about 2 s and 40 studs |
| Window 3×2 from 30 studs, new player | Succeeds within a few tries |
| Landing on a table | Within 2–3 tries |

Playtest on PC, phone and gamepad before moving to networking ([12-roadmap.md](12-roadmap.md), phase 2d).

## Network implications

- The client sends `{ t, cf, vel, throttle }` at 30 Hz ([03](03-network.md)).
- The server checks the displacement against the speed cap, compares the reported velocity with the actual displacement, ray-checks walls, and runs fuel on PRO ([11-math §15](11-math.md)).
- Other players' rockets use cubic Hermite interpolation with velocity, 100 ms in the past ([§15.7](11-math.md)). Their flame, nozzle ring, engine light and jet wash are driven by the replicated `throttle`.
- A `Correct` from the server snaps the local rocket back to the last valid state and keeps its velocity. It should be rare; log it in Studio.
