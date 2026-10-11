# 11 — Math reference

Every formula the game uses, with its variables, default values and the reason it exists.
All numbers are starting points: they live in `ReplicatedStorage.Shared.FlightTuning` / `Config` and get tuned in play (see [05-flight.md](05-flight.md#tuning)). Flight defaults are calibrated against the reference video: see [13-reference-physics.md](13-reference-physics.md).

**The flight model in one sentence:** the player rotates the rocket at any time with WASD, Space pushes it along its nose, and the velocity stays the velocity until thrust, gravity, drag or a surface changes it. Inertia is the core of the feel; nothing aligns the velocity with the nose.

## Notation

| Symbol | Meaning |
|---|---|
| `p`, `v` | rocket position (studs), velocity (studs/s) |
| `f` | unit nose direction (`Root.CFrame.LookVector`) |
| `ω` | angular velocity of the rocket (vector, rad/s) |
| `up` | world up `(0, 1, 0)` |
| `camFwd`, `camRight` | camera look and right vectors, flattened onto the horizontal plane and normalised |
| `dt` | fixed simulation step, `1/60` s |
| `a·b`, `a×b` | dot product, cross product |
| `\|a\|`, `â` | length, unit vector (`a.Unit`) |
| `lerp(a, b, t)` | `a + (b − a)·t` |
| `saturate(x)` | `clamp(x, 0, 1)` |
| `vmax` | top speed from the Speed upgrade (§14) |
| `vf = v·f` | forward speed (along the nose) |

---

## 1. Time stepping

**1.1 Fixed step with render interpolation**
```
acc += min(frameDt, 0.25)
while acc >= dt:
    prev = curr
    curr = step(curr, dt)
    acc -= dt
alpha = acc / dt
renderCFrame = prev.cframe:Lerp(curr.cframe, alpha)
```
Physics runs at exactly 60 Hz on every device, so the rocket behaves the same at 30 or 240 FPS. Interpolating between the last two states removes stutter on 120/144 Hz screens. The 0.25 s clamp stops a hitch from triggering hundreds of catch-up steps.

**1.2 Frame-rate independent smoothing**
```
x ← x + (target − x) · (1 − e^(−dt/τ))
```
`τ` is a time constant: after `τ` seconds, `x` has covered 63% of the distance to `target`. Used for throttle, FOV and camera distance. Unlike `lerp(x, target, 0.1)`, it gives the same result at any frame rate.

---

## 2. Attitude input: WASD relative to the camera

**2.1 Input axes**
```
pitchIn = W − S          (−1..1; gamepad / mobile stick Y)
turnIn  = D − A          (−1..1; stick X)
```

**2.2 Rotation axes**
```
axisWS = −camRight                                 W tips the nose away from the camera (down when flying level)
k      = f × camRight                              |k| = sin(angle between the nose and the camera's right)
axisAD = (k + (1 − |k|) · (−up)).Unit              D moves the nose right on screen
```
- W/S rotate the rocket about the camera's right axis. Upright, that leans it forward or back; flying level away from the camera, it dives or climbs; holding W keeps flipping it forward.
- A/D move the nose left or right **on screen**, whatever the rocket's orientation: upright it leans sideways, flying level it turns, upside down or facing the camera it still goes the way the key says.
- Edge case: when the nose points straight along the camera's right or left, `k` vanishes and A/D fall back to a turn about world up (A turns a right-pointing nose away from the camera).
- Both axes come from the camera's yaw only, so "W" always means "away from me" whatever the camera pitch.

**2.3 Target angular velocity**
```
ωtarget = ωmax · (pitchIn · axisWS + turnIn · axisAD)          ωmax = 12 rad/s (≈ 690°/s)
```
If both inputs are held, the vector is clamped to length `ωmax`.

---

## 3. Throttle and fuel

**3.1 Engine spool**
```
throttle ← throttle + (input − throttle) · (1 − e^(−dt/τ))      τ = 0.05 s rising, 0.08 s falling
input = 1 while Space is held (or the mobile thrust button);  input = trigger^1.5 on gamepad RT
```
Crisp: the reference uses taps as short as 0.1 s, and each one has to register as a distinct push.

**3.2 Ignition kick.** When `throttle` crosses 0.05 going up:
```
v ← v + f · 4
```
A small instant shove so every tap feels like a punch.

**3.3 Fuel (PRO servers only)**
```
fuelMax = 3.0 + 0.5 · fuelLevel                                  (seconds of full thrust)
flying:    fuel ← max(0, fuel − throttle · dt)
standing:  fuel ← min(fuelMax, fuel + 0.6 · dt)                   (after 0.3 s standing still)
fuel = 0  ⇒  input forced to 0
```
Fuel refills only while standing, and standing is the disguise pose. This ties "refuel" to "hide".

**3.4 Low-fuel sputter** (fuel below 15%). Every step, with probability
```
P = 6 · dt · (1 − fuel / (0.15 · fuelMax))
```
the throttle target drops to 0 for 0.05 s. The engine coughs more as the tank empties: a warning you can hear and feel.

---

## 4. Rotation dynamics: always available, with angular inertia

**4.1 Spin up and brake**
```
Δω = ωtarget − ω
rate = (|ωtarget| > |ω|) ? αup : αbrake                 αup = 90 rad/s²,  αbrake = 120 rad/s²
ω ← ω + Δω̂ · min(|Δω|, rate · dt)
```
- Rotation works **in every state**: on the pad, coasting with the engine off, mid-thrust. There is no "steering authority" tied to throttle or speed.
- Holding a key reaches full spin in about 0.13 s; a 180° flip takes about 0.38 s. A 0.05 s tap moves the nose about 10°: fine control comes for free.
- Releasing the keys brakes to zero in about 0.1 s and **the rocket holds its attitude**. Nothing levels it out in the air and nothing points it along its flight path.

**4.2 Integrate the orientation**
```
rot ← CFrame.fromAxisAngle(ω̂, |ω| · dt) * rot      (re-orthonormalise every step)
f   = rot.LookVector
```

**4.3 Standing limits** (state `Standing`, [§8](#8-landing-and-standing))
```
tilt = angle(f, up),  limited to ≤ 35°, pivoting on the base (§8.3)
no input and no thrust  ⇒  spring back to upright (§8.2)
```
On the ground WASD aims the takeoff instead of tipping the rocket over.

---

## 5. Linear forces

All accelerations are added up, then `v ← v + Σa · dt`. Rotation never touches `v`.

**5.1 Gravity**
```
a_g = (0, −g, 0)          g = 110 studs/s²
```
Strong enough for the snappy hops seen in the reference, still lighter than Roblox's 196.2. This is the least certain number: [13-reference-physics.md](13-reference-physics.md#gravity) explains why, and gives the presets to compare in the flight gym.

**5.2 Thrust with taper**
```
a_t = f · T · throttle · (1 − saturate(max(0, vf) / vmax)²)          T = 360 studs/s²
```
- Thrust-to-weight `T/g ≈ 3.3`, derived from the reference's hop rhythm.
- A 0.13 s tap from standing lifts the rocket about 1 rocket length; hovering upright needs about 31% throttle (rapid tapping works too).
- Full push from a standstill, fading to zero as the forward speed reaches `vmax`: top speed arrives softly.
- Only speed **along the nose** is tapered, so turning the rocket and thrusting the other way always brakes at full power.

**5.3 Air drag**
```
a_d = −v · (c1 + c2 · |v|)          c1 = 0.10 /s,  c2 = 0.002 /stud
```
Deliberately low: momentum carries. At 40 studs/s the slowdown is only 7 studs/s². To stop, you turn the nose against your motion and burn, exactly like the reference does when it reaches the house.

**5.4 Ground cushion.** Ray from the nozzle along `−f`, 8 studs long; it hits at distance `h` with surface normal `n`:
```
j   = saturate(f·n)                                  1 when the jet blasts straight into the surface
a_c = n · T · 0.15 · throttle · (1 − h/8)² · j
```
A jet blowing into a nearby surface pushes the rocket away from it, which makes landings and low hovering easier. Small on purpose: it softens, it does not auto-hover.

---

## 6. Speed cap

```
vcap = 1.5 · vmax + 20
if |v| > vcap:  v ← v̂ · vcap
```
An absolute limit that leaves room for dives and falls (a 30-stud fall reaches about 81 studs/s). The server validates movement against it (§15).

---

## 7. Collisions: collide and slide

**7.1 Sweep**
```
Δ = v · dt
N = p + f · (Length/2 − 0.45)                          nose sphere centre
spherecast(N, 0.45, Δ) and spherecast(p, 0.45, Δ)      take the earliest hit: distance t, normal n
```
Two spheres (nose and centre) cover the long body well enough, and the sweep is continuous, so no tunnelling at any speed.

**7.2 Move up to the contact**
```
p ← p + Δ̂ · max(0, t − ε)          ε = 0.01
```

**7.3 Impact values**
```
vn     = −(v·n)          speed into the surface (positive = hitting it)
noseIn = −(f·n)          1 = nose pointing straight into the surface
```

**7.4 Crash** (explosion, §13):
```
noseIn > cos 35° (0.82)   and   vn > 45   and   timeSinceTakeoff > 0.15 s
```
Only a hard, nose-first hit explodes. Glancing hits slide. The first 0.15 s after takeoff are forgiven.

**7.5 Landing**
```
n.Y > 0.7   and   |v| < 25   and   throttle < 0.1   and   tilt < 60°      ⇒  state Landing (§8)
```

**7.6 Slide response**
```
vt = v + n · vn                              tangential part of v
if vn > 2:   v ← vt · (1 − μi) + n · (e · vn if vn > 8 else 0)      impact friction μi = 0.3, bounce e = 0.15
else:        v ← vt · e^(−μs · dt)                                    sliding contact, μs = 2 /s
Δrest ← (Δ − Δ̂ · t);  Δrest ← Δrest − n · (Δrest·n)                 repeat the sweep with the rest, at most 3 times
```
An impact costs speed once and can bounce slightly. Continuous sliding loses speed over time rather than per step, so the result does not depend on the step rate. A rocket skimming the floor tilted (as in the reference) slides on its fins until the next tap.

**7.7 Bonk feedback.** If `vn > 8`: `trauma += 0.5 · saturate(vn / 40)` (§10.6), plus a bonk sound.

---

## 8. Landing and standing

**8.1 Upright target**
```
flat = (f − up · (f·up)).Unit            current heading, flattened (camera forward if degenerate)
rotUp = CFrame.lookAt(Vector3.zero, up, flat)
```
The nose points up and keeps its heading.

**8.2 Settle: tilt spring**
```
ψ = angle between f and up,   k = (f × up).Unit
α = ωs² · ψ · k − 2 · ζs · ωs · ω          ωs = 16 rad/s,  ζs = 0.8
```
The reference rights itself fast and cleanly: 25° → upright in about 0.13 s, no visible wobble. `ζs = 0.8` gives under 2% overshoot and settles in about 0.3 s. Lowering `ζs` to about 0.4 brings back a bottle-like wobble if playtests want more charm.

**8.3 Pivot at the base**
```
p = B + f · (Length / 2)          B = contact point under the base
```
The rocket tilts around its base, not its centre, so it never slides or sinks while settling or while WASD aims the takeoff.

**8.4 Disguised.** `Standing` for 0.3 s with `|ω| < 0.5` rad/s and no tilt input.

**8.5 Support check.** Ray straight down from the base, 0.6 studs. No hit ⇒ state `Falling`.

**8.6 Takeoff.** Thrust while `Standing` leaves the ground along the current (possibly WASD-tilted) nose. For the first 0.15 s:
```
s  = t / 0.15
ft = lerp(up, f, s).Unit              thrust direction used in 5.2
```
A takeoff tilted toward the floor lifts first, instead of ploughing into the ground.

---

## 9. Visual-only motion

**9.1 Engine vibration** (applied to the rendered CFrame only)
```
offset = 0.03 · throttle · (noise(t·40, 1), noise(t·40, 2), 0)        studs, Root local X/Y
```

---

## 10. Camera: free orbit, rigid follow

**10.1 Orbit (player-controlled, independent of the rocket's attitude)**
```
yaw   ← yaw − Δx · sens
pitch ← clamp(pitch − Δy · sens · (invertY ? −1 : 1), −80°, 80°)
camRot = CFrame.Angles(0, yaw, 0) * CFrame.Angles(pitch, 0, 0)
PC: right mouse drag (Roblox-style) or Shift to lock the mouse; gamepad: right stick; mobile: drag on the right half
touch:  Δ' = sign(Δ) · |Δ|^1.3 · sensTouch
```

**10.2 Desired position**
```
focus = P + up · 1.2 · Length            look slightly above the rocket, so it sits in the lower third of the screen
dist  ∈ [8, 25] studs, default 12        mouse wheel / pinch zoom
C*    = focus + camRot · (0, 0, dist)
```

**10.3 Follow: near-rigid critically damped spring** (frame-rate independent form, `smoothTime = 0.04 s`)
```
w      = 2 / smoothTime
x      = w · dt
e      = 1 / (1 + x + 0.48x² + 0.235x³)
change = C − C*
temp   = (vel + w · change) · dt
vel    = (vel − w · temp) · e
C      = C* + (change + temp) · e
```
In the reference the rocket stays at a fixed screen spot to within about 2 px: the camera is effectively rigid. 0.04 s adds a hint of weight without visible lag.

**10.4 Wall avoidance**
```
spherecast(focus, 0.3, C* − focus)  ⇒  hit at distance t
allowed = t − 0.2
dist = allowed                               if allowed < dist   (pull in instantly)
dist ← smooth(dist, allowed, τ = 0.25)       otherwise           (ease back out)
```

**10.5 FOV**
```
fov ← smooth(fov, 70 + 10 · saturate(|v| / vmax), τ = 0.2)
on ignition: fov += 3 · e^(−t / 0.1)
```
Not in the reference: our addition to sell speed. Keep it subtle.

**10.6 Shake: trauma model**
```
trauma  ← saturate(trauma + impulse);   trauma ← max(0, trauma − 1.2 · dt)
steady  = 0.15 · throttle + 0.35 · I_wash                (I_wash from §11.2)
s       = max(trauma, steady)²
offset  = s · 0.25 · (noise(t·25, 1), noise(t·25, 2))      studs
tilt    = s · 2° · noise(t·25, 3)
impulses: landing 0.25 · saturate(vn/25),  bonk 0.5 · saturate(vn/40),  explosion 1 − dist/40
```
Squaring makes small shakes subtle and big ones violent.

---

## 11. Jet wash: exhaust spreading over a surface

**11.1 Probe.** Ray from the nozzle along `−f`, `Rw = 8` studs ⇒ hit point `H`, normal `n`, distance `h`.

**11.2 Intensity**
```
j = saturate(f·n)
I = throttle · (1 − h / Rw)² · (0.3 + 0.7 · j)
```
Strongest when the jet is close and blasting straight in. A grazing jet still kicks up some dust.

**11.3 Rig placement**
```
t1  = (n × (|n.Y| < 0.99 ? up : (1,0,0))).Unit          any vector lying in the surface
rig = CFrame.fromMatrix(H + n · 0.05, t1, n)             rig +Y = surface normal
```
The rig carries 8 attachments at `θk = k · 45°` around the normal. Each attachment's +Y is the in-plane direction `rk`.

**11.4 Tilt bias**
```
jp = (−f) − n · ((−f)·n)                  jet direction projected onto the surface
s  = saturate(|jp| / 0.7)                 0 = straight down, 1 = strongly tilted
w_k = 1 + s · (rk · ĵp)                    weight per direction, averages to 1
rate_k = I · 60 · w_k                      particles/s per direction
```
A tilted rocket blows most of the dust to the side it is leaning toward. The total amount stays the same.

**11.5 Particle motion**
```
speed = lerp(15, 30, I),   Drag = 4,   lifetime 0.3–0.6 s,   small upward Acceleration for buoyancy
```

**11.6 Dust colour**
```
dust = lerp(surfaceColor, grey(0.55), 0.35)
ColorSequence: 0 → orange, 0.15 → orange, 0.3 → dust, 1 → dust
```
Fire at the impact point, then dust tinted by whatever surface was hit (green on grass, grey on rock).

**11.7 Scorch marks**
```
heat[cell] += I · dt                 cell = surface position snapped to a 2-stud grid
heat > 1.5  ⇒  place a scorch plate, heat = 0;  plates fade out over 5 s
```

---

## 12. Guards

**12.1 Patrol sweep**
```
yaw(t) = yaw0 + A · sin(ωp · t)          A = 60°,  ωp = peakSpeed / A = (30°/s) / 60° = 0.5 rad/s
```
A sine sweep slows down at the ends, the way a turret looks around.

**12.2 Lock-on**
```
when max suspicion > 0.35:   yaw ← yaw + clamp(Δyaw_to_rocket, −90°·dt, +90°·dt)
when suspicion < 0.1 for 2 s:  resume the sweep from the current angle
```
This is the "turret turns straight at you" moment from the reference.

**12.3 Vision test** (guard `g`, rocket `r`)
```
to = Pr − Pg
inRange = |to| ≤ range
inCone  = to.Unit · look ≥ cos(FOV / 2)
clear   = raycast(Pg, to) against map geometry hits nothing closer than |to| − 0.5
visible = inRange and inCone and clear              (test the rocket centre and nose; either counts)
```

**12.4 Moving test** (server side)
```
speedS = |Pr(now) − Pr(now − 0.2 s)| / 0.2
moving = throttle > 0.05  or  speedS > 2
```
The 0.2 s window absorbs network jitter, so a rocket that has just stopped is not caught by a stale sample.

**12.5 Suspicion**
```
m        = 1.0 on Normal servers, 1.5 on PRO servers
distFac  = lerp(1.0, 0.5, saturate((|to| − 0.5·range) / (0.5·range)))      farther away = noticed more slowly
visible and moving:  s += 1.5 · m · distFac · dt
visible and still:   s += 0.25 · (1 − 0.04 · disguiseLevel) · m · distFac · dt
not visible:         s −= 0.5 · dt
s = saturate(s);     s = 1  ⇒  Spotted (the guard fires)
```
Moving in plain view gets you caught in about 0.7 s. Standing still in view buys about 4 s at disguise 0 and about 20 s at disguise 20.

**12.6 Stealth tracking**
```
peak = max(peak, s)      over the whole run, across all guards
stealthStrike = peak < 0.5
```

---

## 13. Explosions, damage, rewards

**13.1 Warhead**
```
W  = 100 · 1.12^warheadLevel          damage
Rb = 6 + 0.1 · warheadLevel           blast radius (studs)
```

**13.2 Damage to a target**
```
d   = distance from the explosion to the closest point of the target's hitbox
dmg = W · saturate(1 − d / Rb)
```
A direct hit deals full damage, falling off linearly to zero at the blast edge.

**13.3 Target health**
```
HP: Tank 100,  Heavy tank 250,  Bunker 600,  Summit radar (PRO) 1500
regen: +5% maxHP per second after 10 s without damage
```
One rocket kills a target only when the Warhead is high enough: Heavy tank from level 9, Bunker 16, Radar 24. Below that, players chip away at it, alone over several runs or together.

**13.4 Damage share**
```
share_i = dmg_i / Σ dmg          sums over damage dealt in the last 60 s before the kill
```

**13.5 Cash and XP per kill**
```
cash_i = round(base · (1 + 0.5 · rebirths_i) · (stealth_i ? 1.5 : 1) · (PRO ? 2 : 1) · share_i)
xp_i   = round(xpBase · (stealth_i ? 1.5 : 1) · (PRO ? 1.5 : 1) · share_i)
base / xpBase:  Tank 50 / 10,  Heavy tank 250 / 25,  Bunker 1200 / 60,  Summit radar 6000 / 150
participation:  +2 XP for every finished run, even when spotted
```

---

## 14. Progression

**14.1 XP needed for the next level**
```
need(L) = round(25 · L^1.35)
```
Levels 1→10 need about 2,100 XP in total, roughly 45–60 minutes of play.

**14.2 Level-up loop** (`xp` stores progress inside the current level)
```
while xp ≥ need(level):  xp −= need(level);  level += 1
```

**14.3 Upgrade cost**
```
cost(type, L) = round(base_type · 1.35^L)          base: Speed 25, Disguise 40, Warhead 50, Fuel 30
```

**14.4 Upgrade effects**

| Upgrade | Effect | Cap |
|---|---|---|
| Speed | `vmax = 40 + 2.5 · L` | L 15 → 77.5 |
| Disguise | still-in-view suspicion × `(1 − 0.04 · L)` | L 20 → × 0.2 |
| Warhead | `W = 100 · 1.12^L`, `Rb = 6 + 0.1 · L` | L 30 |
| Fuel (level 10+, matters on PRO) | `fuelMax = 3 + 0.5 · L` | L 14 → 10 s |

**14.5 Rebirth**
```
cost(R) = 10,000 · 2.5^R          resets cash and upgrades; keeps level, XP and skins
multiplier = 1 + 0.5 · R
```

---

## 15. Networking and validation

**15.1 Client state stream:** 30 Hz, `{ t, cf, vel, throttle }`, where `t = workspace:GetServerTimeNow()` at send time.

**15.2 Displacement bound**
```
Δt   = clamp(serverRecvTime − lastRecvTime, 1/60, 0.5)
dmax = vcap · 1.2 · Δt + 1.0
violation if |p_new − p_lastValid| > dmax
```

**15.3 Velocity consistency**
```
violation if |vel − (p_new − p_last)/Δt| > 0.5 · vcap + 10          (after 3 consecutive mismatches)
```

**15.4 Wall check**
```
raycast(p_lastValid, p_new − p_lastValid) against map geometry
violation if it hits closer than |p_new − p_lastValid| − 0.5    ⇒  Correct(p_lastValid)
```

**15.5 Violation bucket**
```
score += 1 per violation;   score ← max(0, score − 0.5 · dt)
score ≥ 5  ⇒  end the run with reason Rejected
```
Equivalent to "5 violations within roughly 10 s". A single lag spike never ends a run.

**15.6 Server-side fuel** (PRO)
```
fuelS ← fuelS − throttle · Δt  (refill while standing, same rule as 3.3)
violation if fuelS ≤ 0 and throttle > 0.05 for more than 0.3 s
```

**15.7 Remote rocket interpolation** (cubic Hermite, using velocity)
```
tr = workspace:GetServerTimeNow() − 0.1                 render 100 ms in the past
find snapshots s0, s1 with t0 ≤ tr ≤ t1
u  = (tr − t0) / (t1 − t0),   D = t1 − t0
p(u) = (2u³ − 3u² + 1)·p0 + (u³ − 2u² + u)·D·v0 + (−2u³ + 3u²)·p1 + (u³ − u²)·D·v1
rot(u) = r0:Lerp(r1, u)
tr > t_newest (packet loss):  p = p1 + v1 · (tr − t1) for at most 0.2 s, then hold
```
Using velocity gives smooth curves through the samples instead of a zig-zag polyline. That matters for fast, arcing rockets.

**15.8 Timestamp sanity** (server, per received state)
```
t ← clamp(t, recvTime − 0.5, recvTime)
```

---

## 16. Hit tests (server)

**16.1 Swept rocket vs sphere target**
```
seg = P1 − P0                                    rocket movement this sample
u   = clamp(((C − P0)·seg) / |seg|², 0, 1)
Q   = P0 + u · seg                               closest point on the path
hit = |C − Q| ≤ Rt + 0.5
```

**16.2 Point vs box hitbox** (oriented box with CFrame `B`, half-size `h`)
```
q  = B:PointToObjectSpace(Q)
c  = (clamp(q.X, −h.X, h.X), clamp(q.Y, −h.Y, h.Y), clamp(q.Z, −h.Z, h.Z))
d  = |q − c|                                     0 when inside; this is the d used in 13.2
```
