# 00 — Overview

## Pitch

You are a rocket that hides in plain sight. Launch from a cliff pad, fly through a rocky valley, slip through a window into a guarded outpost. When a guard's turret swings toward you, land, stand upright and become a flower vase. When it looks away, strike.

## Core loop

```
LAUNCH → fly to a site → infiltrate → freeze when watched → strike → cash + XP → upgrades / skins → LAUNCH
```
A run lasts 30–90 seconds. Failing is quick and cheap (you get spotted, you crash), so the next run is one click away.

## Pillars

1. **Flight feel first.** Pure inertia: rotate any time with WASD, push with Space, plan your stops. Snappy hops, momentum that carries, a camera that follows tightly, a jet wash that reacts to every surface. If flying is not fun on its own, nothing else matters. See [05-flight.md](05-flight.md).
2. **Hide in plain sight.** Moving in a guard's view gets you caught in under a second. Standing still makes you a "vase". The tension of a turret looking straight at you, and not noticing, is the signature moment.
3. **Short runs, fast progression.** Levels, upgrades and rebirths in the style of a simulator game: always something to buy next.
4. **Stud style.** Classic Roblox look: studded surfaces, flat colours, effects made of squares and cubes.

## Modes

| | Normal servers | PRO servers (level 10+) |
|---|---|---|
| Who | Everyone | Players at level 10 or above |
| Fuel | None: thrust is unlimited | Yes: refills only while standing |
| Guards | Base profile | Wider view, suspicion builds faster |
| Rewards | ×1 | Cash ×2, XP ×1.5 |
| Content | Zone sites | Plus PRO-only sites |

Fuel is too hard for younger players, so it is a PRO feature. Details: [08-pro-servers.md](08-pro-servers.md).

## Final decisions

| Topic | Decision | Why |
|---|---|---|
| Framework | Branch v2.5.2: one `Main` Script / LocalScript per side, systems as Service/Controller segments | Ordered `Init`/`Start` with `Dependencies` removes the load-order races of a many-scripts setup; compiled, validated networking. This replaces the earlier "multi-script" idea. |
| Game mode | PvE, guards are bots | Faster to ship; a PvP prop-hunt variant is in the backlog |
| Map | One place with a hub and zones, unlocked by level | Instant start, no teleports inside Normal play |
| Platforms | Mobile first, plus PC and gamepad through `Branch.Keybinds` | Main audience of this genre |
| Rocket movement | Kinematic, simulated on the client, validated by the server | Zero input latency, no physics jitter, full anti-cheat control |
| Replication | Custom, on top of Branch networking (`Unreliable`, batched snapshots, Hermite interpolation) | One non-humanoid entity per player; Branch covers transport. Chrono is not needed for now; replication sits behind one service/controller pair so it can be swapped. |
| Controls | WASD rotates the rocket relative to the camera (always available), Space is thrust, free orbit camera | Confirmed against the reference ([13](13-reference-physics.md)); inertia is the core of the feel |
| PRO | Separate place, open from level 10 | Difficulty tier for experienced players |
| First zone | Rocky Valley (rocky nature) | Matches the first asset pack |
| Monetization, tutorial | After the core game is complete | See [backlog.md](backlog.md) |

## Glossary

| Term | Meaning |
|---|---|
| Run | One launch, from the pad until the rocket explodes, is shot down or times out |
| Zone | A map area (Zone 1 = Rocky Valley). "Level" always means player level. |
| Site | A guarded location inside a zone with one or more targets |
| Sentinel | A target that also watches (the tank in the reference) |
| Watcher | A guard that is not a target (turret, searchlight tower) |
| Disguised | The rocket has stood upright and still for 0.3 s |
| Jet wash | Exhaust spreading over a surface the jet hits |
