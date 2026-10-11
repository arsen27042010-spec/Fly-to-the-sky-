# 13 — Reference physics analysis

A frame-by-frame measurement of how the rocket in [`reference/reference-video.mp4`](../reference/reference-video.mp4) actually moves, and what that means for our flight model. The flight defaults in [11-math.md](11-math.md) are calibrated from this document.

## Method and limits

- **Source:** 612 frames at 30 fps (20.4 s). The game image occupies rows 290–825 of the 576×1024 vertical video, so the analysed area is 576×536 px. It is a TikTok crop of a landscape recording; the original field of view and zoom are unknown.
- **Automated, per frame:**
  - flame pixels (bright saturated orange/yellow);
  - warm "engine light" glow on the floor;
  - fin and stem colour masks;
  - landmark tracking (house width, TV picture position);
  - frame differences.
- **Manual:** dense frame sheets for every key moment, every 1–2 frames.
- **Raw data:** per-frame flame pixels, floor glow, thrust detection and fin position are in [`reference/physics-measurements.csv`](../reference/physics-measurements.csv).
- **Scale:** the video has no absolute scale. Distances are expressed in **rocket lengths (RL)** and converted with our rocket (1 RL = 4.83 studs). Wherever a field of view is needed, Roblox's default vertical FOV of 70° over the 536-px height is assumed (`f ≈ 383 px`).
- **Confidence** is marked on every number: **High** (directly visible), **Medium** (measured with one assumption), **Low** (inferred).

## 1. Control model

| Finding | Evidence | Confidence |
|---|---|---|
| WASD rotates the rocket, Space is thrust | Confirmed by the user. The "x" cursor sits at (310, 130) px for all 612 frames: the mouse never steers. | High |
| Rotation is available at all times, engine on or off | 19.13–19.33 s: a full 180° flip mid-air with no flame; 4.67–4.87 s: a 120° flip just before landing | High |
| The rocket holds its attitude when no key is pressed | 7.7–8.6 s: tilt held at about 55° across two thrust taps and the coasts between them; 9.2–9.6 s: held at about −40° | High |
| Velocity does not follow the nose (inertia is the core) | 0.4–1.4 s: the nose points right while the rocket keeps flying with the camera; 3.7–4.9 s: nose down and back, burning to brake | High |
| The camera orbits freely and follows position only | The camera keeps looking at the house while the rocket points right (0.4–1.4 s); orbits around the standing rocket (17.6–18.5 s) | High |
| Thrust comes in short taps indoors, long burns outdoors | See §3 | High |

## 2. Event timeline

| Time | What happens | Measurement |
|---|---|---|
| 0.00–1.70 s | Launch, long continuous burn | Flame 2–3 RL long behind the rocket |
| 0.40–1.40 s | Nose points right (side-on to the camera) while flying | Rocket stays centred: the camera follows position, not attitude |
| 1.40–1.73 s | Turns about 90° to face the house, still burning | ~0.33 s, about 270°/s on average |
| 1.80–2.30 s | Burn tapers off | Flame pixels 6,400 → 1,800 |
| 2.30–2.90 s | Coast, engine off | Approach speed kept (§5) |
| 2.67–3.03 s | Tap | 0.37 s |
| 3.40–3.70 s | Pitches over: nose from up-forward to down-back | ~100–130° in ~0.5 s |
| 3.73–4.90 s | Burn with the nose down and back: brakes and descends into the yard | 1.17 s; world-space flame trail left above it |
| 4.67–4.87 s | Flips to nose-up just above the grass | ~120° in 0.2 s, about 600°/s |
| 4.93–5.23 s | Landing burn | 0.30 s |
| 5.30–6.20 s | Stands upright on the grass, still | Settles from −14° to about −4° in ~0.3 s, no wobble |
| 6.23–6.40 s | Two taps lift it upright to the window-sill height | 0.03 s + 0.10 s |
| 6.70–6.90 s | Tilts about 30–45° and goes in through the window | — |
| 6.87–10.53 s | Indoors: skims along the floor tilted 40–55°, tapping | 8 taps; see §3 |
| 8.63–8.90 s | Rotates back to upright | ~55° in 0.27 s, about 200°/s |
| 11.20–11.40 s | Tap close to the floor | Strongest floor glow, then white dust cubes at 11.43 s |
| 12.27–13.90 s | Tiny correction taps near the table | 0.03–0.10 s each |
| 14.27–14.40 s | Lands on the table, rights itself | 25° → 0° in 0.13 s, no overshoot |
| 14.40–18.50 s | Stands still, disguised, for about 4 s | Frame difference ≈ 0.8: nothing moves |
| 14.40–14.60 s | Tank turret swings onto the rocket | ~90° in ~0.2 s (≥ 450°/s) |
| 14.60–17.60 s | Turret stares at the rocket | 3.0 s |
| 17.60–17.67 s | Turret swings away | ~90° in ≤ 0.07 s (a snap) |
| 18.60–18.73 s | Vertical takeoff tap | 0.13 s; table lit up, white dust cubes spread across it |
| 18.73–19.10 s | Ballistic hop onto the tank, engine off | Rises about 1 RL, lands about 0.4 s after the tap |
| 19.13–19.33 s | Flips nose-down on top of the tank, engine off | ~180° in ~0.2–0.3 s |
| 19.80–20.40 s | Burns nose-first into the tank | Vertical flame column, big white cube cloud |

## 3. Thrust taps

Detected from flame pixels (outdoors) and floor glow (indoors), thresholds above 2,500 px.

| Indoor tap start | Duration | Gap to the next tap |
|---|---|---|
| 6.87 s | 0.17 s | 0.20 s |
| 7.23 s | 0.40 s | 0.23 s |
| 7.87 s | 0.13 s | 0.37 s |
| 8.37 s | 0.13 s | 0.47 s |
| 8.97 s | 0.17 s (split 0.07 + 0.10) | 0.53 s |
| 9.70 s | 0.17 s | 0.43 s |
| 10.30 s | 0.20 s (split 0.17 + 0.03) | 0.67 s |
| 11.20 s | 0.20 s | then correction taps |

- **Typical tap:** 0.13–0.20 s (median about 0.17 s).
- **Typical gap:** 0.2–0.67 s (median about 0.45 s).
- **Rhythm:** one tap roughly every 0.6 s.
- **Duty cycle indoors:** about 35%.
- During the 7.23 s tap the floor glow grows from 2,500 to 23,000 px. The nozzle is closing in on the floor while thrusting, so taps catch the rocket's fall rather than launching it from rest. *(High)*
- Outdoors the player holds thrust for 0.4–1.7 s at a time. *(High)*

## 4. Rotation

| Event | Change | Time | Mean rate | Engine |
|---|---|---|---|---|
| Turn toward the house | ~90° | 0.33 s | ~270°/s | On |
| Pitch-over to brake | ~100–130° | ~0.5 s | ~200–260°/s | Partly |
| Flip before landing | ~120° | 0.2 s | ~600°/s | Sparse |
| Back to upright indoors | ~55° | 0.27 s | ~200°/s | Off |
| Lean to the other side | ~40° | 0.4–0.6 s | ~80–100°/s | Off |
| Self-righting on the table | ~25° | 0.13 s | ~190°/s | Off (landing) |
| Flip nose-down on the tank | ~180° | 0.2–0.3 s | ~600–900°/s | Off |

Small corrections are slow and big flips are fast. That is what angular inertia looks like: the longer a key is held, the faster the rocket spins. Our model therefore ramps the spin up (`αup = 90 rad/s²`) toward `ωmax = 12 rad/s ≈ 690°/s`, and brakes quickly on release (`αbrake = 120 rad/s²`). *(Medium)*

## 5. Translation and inertia

### Cruise speed (outdoors)

The house's apparent width grows as the rocket approaches it. The distance in house widths is `D/W = f / width_px`:

| t | House width (px) | D/W |
|---|---|---|
| 1.0 s | 120 | 3.19 |
| 1.4 s | 136 | 2.82 |
| 1.8 s | 152 | 2.52 |
| 2.2 s | 180 | 2.13 |
| 2.4 s | 198 | 1.93 |
| 2.6 s | 229 | 1.67 |
| 2.8 s | 266 | 1.44 |

- Closing speed from 1.8 to 2.8 s: about 1.1 house widths per second. It does not drop during the 2.3–2.9 s coast, so drag is low.
- Assuming a 25–35-stud-wide house, that is **about 27–38 studs/s**, roughly 6–8 RL/s. *(Medium; it scales with the unknown FOV.)*
- Our old top speed of 60 studs/s was about 1.5–2× the reference. The new base is 40 studs/s.

### Braking is done by the player

There is no aerodynamic braking. To stop at the house the rocket pitches nose-down and back and burns for 1.17 s. Speed only changes through thrust, gravity, a little drag and contact. *(High)*

### Thrust-to-weight

For a tap of length `tb` at tilt `θ` with airtime `τ` back to the same height:
```
Δv_up = (T·cosθ − g)·tb = g·τ/2     ⇒     T/g = (1 + τ / (2·tb)) / cosθ
```
With the indoor values (`tb ≈ 0.15 s`, `τ ≈ 0.4 s`, `θ ≈ 45°`): **T/g ≈ 3.3**. Unlike gravity, this ratio does not depend on the unknown scale. *(Medium)*

### Gravity

- The vertical takeoff at 18.60 s (a 0.13 s tap) lifts the rocket about 1 RL. With `T/g = 3.3`, `Δv = 2.3·g·0.13 ≈ 0.3·g`, so the apex is `Δv²/(2g) ≈ 0.045·g`.
- Setting that equal to 1 RL gives `g ≈ 22 RL/s²`, which is **about 107 studs/s²** for our rocket. *(Low–Medium)*
- Cross-check: the 0.4 s airtime of indoor hops at that gravity gives apexes of about 0.45 RL. That fits a rocket that always looks close to the floor.
- Conflicting hint: during the 2.3–2.9 s outdoor coast the horizon barely moves vertically. That fits the rocket being near the top of a climb, but it could also hint at lower gravity.
- **Default 110**, with presets to compare in the flight gym: **80 / 110 / 150 / 196**, always with `T = 3.3·g`.

## 6. Landing and standing

- **Self-righting:** 25° → upright in 0.13 s on the table; −14° → −4° in about 0.3 s on the grass. No visible overshoot or wobble: an almost critically damped spring. *(High)*
- **Standing:** the rocket stands perfectly still on its fins (4 s on the table, about 0.9 s on the grass). *(High)*
- **Takeoff** from standing is vertical and immediate. *(High)*
- **Skimming:** indoors the rocket travels tilted 40–55° close to the floor without righting itself. It either skims on its fins or makes low hops; the video cannot tell which. Landing requires a near-upright attitude and no thrust. *(Medium)*

## 7. Camera

- **Rigid follow:** between taps the rocket's screen position stays constant to within about 2 px (for example 7.1–8.3 s: fins at y = 423–426). The camera follows the position with essentially no lag. *(High)*
- **Placement:** the rocket sits in the lower third of the frame (its centre at about 65–70% of the height). The camera looks at a point above the rocket. *(High)*
- **Free orbit:** player-controlled and independent of the rocket's attitude; the cursor never moves, which suggests a right-drag orbit as in Roblox's default camera. *(High)*
- **Distance:** about 1.5–3 RL (it depends on the unknown crop). *(Low)*
- **FOV:** no speed-based FOV change is visible. Our FOV kick is an addition of ours.

## 8. Effects (for [06-effects.md](06-effects.md))

- **Flame:**
  - world-space square particles, 1–2× the body diameter;
  - they grow and darken from yellow to orange to red, living about 0.3–0.5 s;
  - held thrust leaves a 2–3 RL trail, a short tap gives 1–3 squares;
  - squares left behind by a turn stay where they were emitted (0.4–1.7 s).
- **Nozzle ring:** a bright yellow ring at the nozzle exit while thrusting (0.0 s, 1.3 s, 10.4 s, 20.2 s).
- **Engine light:** strong warm light on nearby surfaces, with a glow radius of about 1–2 RL. It gets much brighter as the nozzle nears the surface (floor glow ×9 during one tap). This is the clearest thrust cue indoors.
- **Jet wash:** white/grey cubes spreading over the surface on near-surface taps (11.43 s, 18.67 s).
- **Engine cut:** sparse small orange squares linger and drift (3.8–4.1 s).
- **Strike:** a large cloud of white cubes plus a vertical flame column.

## 9. Guard behaviour

- **Turret:** snaps onto the rocket about 0.1–0.2 s after it lands (≥ 450°/s), stares for 3 s and snaps away (≤ 0.07 s). In the reference this is probably another player or a simple script.
- **Our guards:** we deliberately keep them slower (lock-on 90°/s, [07](07-guards-targets.md)) so the danger can be read. Raising it toward 180°/s is fine if they feel too sluggish.
- **The strike itself:** the rocket **perches** on the tank first, then flips nose-down and burns into it. Suggestion for [07](07-guards-targets.md): a gentle landing on a target counts as perching. The explosion comes from a nose-first contact faster than about 8 studs/s, from thrusting into the target, or from Detonate. This keeps the comedic "standing on the tank" moment.

## 10. What changed in our model

| Aspect | Before | Now | Why |
|---|---|---|---|
| Controls | Camera aims, the nose chases the aim point | WASD rotates the rocket relative to the camera, Space is thrust, free orbit camera | §1 |
| Steering authority | `0.35 + 0.65·throttle` | Always full | Flip with the engine off |
| Weathervane | Nose followed the flight path | Removed | Tilt held for seconds |
| Grip and fin lift | Velocity turned toward the nose; lift ∝ speed² | Removed (optional assist in the backlog) | Inertia is the core |
| Rotation | Spring toward a target direction | Rate control with angular inertia: `ωmax 12`, `αup 90`, `αbrake 120` | §4 |
| Gravity | 60 | 110 | §5 |
| Thrust | 130 (T/g 2.2) | 360 (T/g 3.3) | §5 |
| Top speed | 60 + 3/level | 40 + 2.5/level | §5 |
| Drag | 0.12 / 0.0035 | 0.10 / 0.002 | The coast keeps speed |
| Engine spool | 0.08 / 0.15 s | 0.05 / 0.08 s | 0.1 s taps are distinct |
| Speed cap | 1.35·vmax | 1.5·vmax + 20 | Room for falls |
| Landing settle | ζ 0.4 (wobble) | ζ 0.8, ωs 16 | No wobble in the reference |
| Camera | Locked-mouse aim camera, smooth 0.08 s | Orbit, near-rigid 0.04 s, rocket in the lower third | §7 |
| Visual bank | Auto-roll in turns | Removed | Rotation is direct |
| Ground cushion | 0.25 | 0.15 | Kept subtle |

## 11. Open questions to settle in the flight gym

1. **Gravity preset:** 80 / 110 / 150 / 196, with `T = 3.3·g`. Pick by feel: hops should read as snappy, not floaty.
2. **Skim vs hover indoors:** whether sliding on the fins (§7.6 friction) or low hops feels closer to the reference.
3. **Assists for younger players** on Normal servers, if playtests show the pure-inertia rocket is too hard. Candidates: a gentle auto-level when no keys are pressed, or a small velocity-alignment assist. Both are in the [backlog](backlog.md); PRO servers stay pure.
