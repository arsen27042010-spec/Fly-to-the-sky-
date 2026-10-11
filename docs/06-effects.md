# 06 — Effects and sound

Math for intensity, bias and colours: [11-math.md](11-math.md) §11.

## Stud style rules

- Everything is made of **squares and cubes**: flat colours, no soft gradients, no smoke puffs with blurry edges.
- Palette: yellow `255,220,60` → orange `255,140,30` → red `230,60,30` → smoke `90,90,90`; dust is tinted by the surface (§11.6).
- **One texture to upload:** a 32×32 white square with a thin darker border (the "stud particle"). Put its asset id in `Shared/Config`. As far as we know Roblox has no built-in square particle texture; until it is uploaded, the rocket model uses `fire_main.dds` / `smoke_main.dds` as placeholders.
- The map uses `SurfaceType.Studs` on top faces, classic colours, simple lighting with a bright sky.

## Engine flame (attachment `Exhaust` on the rocket)

| Element | What | Driven by |
|---|---|---|
| `Flame` emitter | Square particles out of the nozzle; length grows with throttle and a little with speed | throttle |
| Flame core | 3–5 pooled neon cubes right at the nozzle, jittering in size: gives the flame volume | throttle |
| `EngineLight` | PointLight, flickers | throttle |
| `Trail` emitter | Grey squares in world space (`LockedToPart = false`), 1–1.5 s life. Shows the arc of flight, which makes the inertia visible. | throttle, speed |
| `Ignition` emitter | `Emit(15)` plus a light flash when the engine starts | ignition event |
| Cut-off puff | `Emit(8)` grey squares when the engine stops | throttle falling below 0.05 |
| Sputter | Flame and light drop out together with the thrust (§3.4) | low fuel (PRO) |

## Jet wash: exhaust spreading over a surface

Every frame while `throttle > 0.1`:
1. Ray from the nozzle along `−f`, 8 studs (§11.1). No hit ⇒ no wash.
2. Intensity `I` from distance and angle (§11.2).
3. **`WashRig`:** an invisible anchored part, moved to the hit point and oriented to the surface normal (§11.3). It carries **8 radial attachments** at 45° steps; each attachment's +Y points outward along the surface and holds its own emitter:
   - `EmissionDirection = Top`, `SpreadAngle ≈ (12°, 20°)`;
   - `Speed = lerp(15, 30, I)`, `Drag = 4`, slight upward `Acceleration`;
   - `Rate = I · 60 · w_k`.
4. **Tilt bias** (§11.4): a leaning rocket blows most of the dust toward the side the jet points to.
5. **Colour** (§11.6): orange for the first 15% of life, then dust tinted by the hit surface's colour. Rebuild the `ColorSequence` only when the surface changes.
6. **Scorch marks** (§11.7): a strong, long blast leaves dark flat plates, pooled, fading over 5 s.
7. **Sound and shake:** a hissing gust with volume = `I`; camera shake grows with `I` (§10.6).
8. **Other players' rockets:** the same ray, using the replicated throttle, only within 150 studs.

Why a rig of 8 emitters instead of reorienting one emitter per burst: whether `Emit` uses the attachment's transform at call time or at render time is not guaranteed, and a fixed rig avoids the question entirely.

## Other effects

| Effect | Description |
|---|---|
| Explosion | Burst of orange/yellow cubes plus grey smoke squares, light flash, debris cubes that bounce (client-only, pooled), shake by distance |
| Landing dust | A small ring of dust squares, tinted by the surface |
| Scrape sparks | Yellow cube sparks along a glancing contact |
| Guard vision cone | A translucent wedge: neutral, turns yellow while suspicion rises, red at lock-on ([07](07-guards-targets.md)) |
| Alert icon | A "?" above the guard while suspicion > 0.35, a "!" when it fires |
| Speed streaks | Thin white blocks streaming past the camera above 85% of top speed (optional, High quality only) |

## Sound

| Sound | Behaviour |
|---|---|
| Engine loop | Pitch rises with throttle and speed; volume = throttle |
| Ignition pop / cut-off "pff" | One-shots |
| Wind | Volume and pitch grow with speed |
| Jet wash hiss | Volume = `I` |
| Landing thunk, scrape, bonk | Volume scales with impact speed |
| Guard: turret servo, "?" sting, "!" sting, shot | From the guard's position |
| Explosion | Volume by distance; muffled when heard from far away |

## Performance

- **No `Instance.new` at runtime.** All parts and attachments come from pools created at startup. Many cubes are moved with `workspace:BulkMoveTo`.
- The player's own rocket gets the full effect set; other rockets get a lighter set (no flame core, fewer particles) and no jet wash beyond 150 studs.
- **Quality setting** (`settings.quality`: Low / Medium / High). On Low:
  - jet wash uses 4 directions instead of 8;
  - no flame core cubes;
  - no speed streaks or scorch marks;
  - particle rates ×0.5.
- Budget: aim for at most ~400 live particles and ~60 pooled cubes on a mid-range phone.
