# 10 — Rocket model

Location: `ReplicatedStorage.Assets.Rockets.Rocket`. Built after the reference: a white body painted with stems and daisies (the "flower vase" disguise), orange fins in an X layout, a grey rounded nose.

## Hierarchy

```
Rocket (Model)          PrimaryPart = Root; attributes Skin = "Flower", Version = 1
├─ Root (Part)          invisible hitbox
│  ├─ Nose (Attachment)     nose tip: hit-test origin
│  └─ Exhaust (Attachment)  nozzle: Flame, Trail, Ignition (ParticleEmitters), EngineLight (PointLight)
├─ Hull (Folder)        body parts
└─ Disguise (Folder)    skin parts (every skin has its own set in Assets/Disguises)
```

## Conventions

- Built **standing**: the nozzle bottom at y = 0, nose up, centred on the Y axis. Body radius `R = 0.5`. Total length `4.83` (about a character's height).
- **Nose = `Root.LookVector`.** Flight sets `Root.CFrame = CFrame.lookAt(p, p + f)`. The standing pose is `CFrame.lookAt(p, p + up, flatForward)`.
- `Root`: Block `1 × 1 × 4.83`, `CFrame = CFrame.new(0, 2.415, 0) * CFrame.Angles(math.rad(90), 0, 0)` (rotating +90° about X points LookVector up).
  - `Anchored = true`, `Transparency = 1`, `CanCollide`, `CanQuery`, `CanTouch` all false.
- **Visual parts:**
  - `Anchored = false`, `Massless = true`, `CanCollide`, `CanQuery`, `CanTouch` all false;
  - each one welded to `Root` (a `Weld` with `C0 = Root.CFrame:Inverse() * part.CFrame`, or a `WeldConstraint`).
  - Only `Root.CFrame` is ever set; everything else follows.
- Roblox `Cylinder` parts run along their X axis. A vertical cylinder: `CFrame.new(pos) * CFrame.Angles(0, 0, math.rad(90))`, `Size = (length, diameter, diameter)`.
- For parts placed around the body at angle φ: outward `n = (cos φ, 0, sin φ)`, tangent `T = (−sin φ, 0, cos φ)`.
- Material: `Plastic` (stud style), smooth surfaces.

## Hull

| Part | Shape | Size | Placement | Colour (RGB) | Material |
|---|---|---|---|---|---|
| Nozzle | Cylinder ↑ | 0.22 × 0.62 × 0.62 | (0, 0.11, 0) | 60, 60, 65 | Metal |
| TailRing | Cylinder ↑ | 0.55 × 1.04 × 1.04 | (0, 0.475, 0) | 235, 120, 40 | Plastic |
| Body | Cylinder ↑ | 3.30 × 1.00 × 1.00 | (0, 2.40, 0) | 225, 225, 220 | Plastic |
| NoseRing | Cylinder ↑ | 0.08 × 1.02 × 1.02 | (0, 4.05, 0) | 170, 170, 175 | Metal |
| NoseBase | Cylinder ↑ | 0.30 × 0.96 × 0.96 | (0, 4.20, 0) | 150, 150, 155 | Plastic |
| NoseTip | Ball | 0.96 | (0, 4.35, 0) | 150, 150, 155 | Plastic |
| Fin1–4 | Block | 0.10 × 0.55 × 0.55 (thickness × height × reach) | radius 0.775, y 0.275 | 235, 120, 40 | Plastic |
| FinSlope1–4 | WedgePart | 0.10 × 0.70 × 0.55 | radius 0.775, y 0.90 | 235, 120, 40 | Plastic |
| FinTip1–4 | Block | 0.12 × 0.18 × 0.12 | radius 1.05, y 0.09 | 235, 235, 235 | Plastic |

- Fins sit at φ = 45°, 135°, 225°, 315°: an X seen from above, as in the last frame of the reference.
- Fin and tip CFrame: `CFrame.fromMatrix(pos, T, up)` (X = thickness along the tangent, Z points at the body axis).
- **FinSlope:** a WedgePart is full height on its +Z side, so this orientation puts the tall side against the body. **Check in Studio:** if the slope faces outward, rotate the wedges 180° about Y.

## Disguise "Flower"

Five daisies, given as (φ, height): (20°, 3.55), (100°, 2.55), (160°, 3.25), (250°, 3.70), (310°, 2.30). Each stem sways ±0.22 rad around the body from bottom to top, alternating per flower; the daisy sits at the stem's top angle.

| Part | Size | Placement | Colour |
|---|---|---|---|
| Stem | 0.05 × L × 0.07 | A straight chord on the body surface (radius 0.52) from y 0.8 at angle φ to the flower at angle φ ± 0.22; `fromMatrix(mid, n_mid, chordDir)` | 76, 153, 60 |
| Leaf ×2 per stem | 0.04 × 0.26 × 0.11 | At 40% and 65% up the stem, offset ±0.08 studs around the body, rotated ±50° about the surface normal, radius 0.525 | 90, 170, 70 |
| Petal ×3 per daisy | 0.03 × 0.38 × 0.08 | Three crossed bars at 0°, 60°, 120° about the normal, radius 0.515 | 245, 245, 240 |
| FlowerCenter | Cylinder 0.06 × 0.14 × 0.14 | `fromMatrix(pos, n, up)`, radius 0.535 | 250, 205, 40 |

About 55 parts in total.

## Attachments and emitters

- **`Nose`:** local `(0, 0, −2.415)` in Root space.
- **`Exhaust`:** local `CFrame.fromMatrix(Vector3.new(0, 0, 2.415), Vector3.xAxis, Vector3.zAxis)`. Its +Y points out of the nozzle, the default emission direction.
- **Emitters** (textures are placeholders until the square particle texture exists, [06](06-effects.md)):

| Emitter | Key settings |
|---|---|
| `Flame` | Disabled by default; colour yellow → orange → red; `LightEmission 1`; size 0.7 → 0.1; lifetime 0.12–0.25 s; rate 150; speed 14–20; spread 10° |
| `Trail` | World space (`LockedToPart = false`); grey; size 0.5 → 1.2; lifetime 1–1.5 s; rate 40; drag 2 |
| `Ignition` | Never enabled, only `:Emit()`; white-yellow → orange; size 1 → 0; speed 20–30; spread 35° |
| `EngineLight` | PointLight, disabled by default; `255,150,60`; range 10; brightness 2 |

## Skins

A skin swaps hull colours (from `Shared/Skins`) and the `Disguise` folder. When a rocket is spawned on a client: clone `Rocket`, replace `Disguise` with `Assets/Disguises/<SkinId>`, weld the new parts to `Root`, recolour the hull.
