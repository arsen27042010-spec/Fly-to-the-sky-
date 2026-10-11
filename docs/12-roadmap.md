# 12 — Roadmap

Work strictly in order. A phase is done only when its acceptance criteria pass. Monetization and the tutorial are **not** in this roadmap ([backlog.md](backlog.md)).

## Phase 0 — Project setup

- Place in `D:\Rocket Game`; Branch 2.5.2 installed through Branch Studio; the Network feature on.
- Folder layout from [02](02-architecture.md); empty segments with `Init`/`Start`; `Shared/Config`, `Shared/FlightTuning`.
- Git for the project; this design repo next to it.
- Greybox of zone 1 ([09](09-zone-1-rocks.md)): hub, pad, the three sites as blocks; a separate test place for the flight gym.

✔ `Branch:GetReport()` shows every segment as `Started`; Branch Studio Problems is empty.

## Phase 1 — Rocket model

- Build the model from [10](10-rocket-model.md) into `ReplicatedStorage.Assets.Rockets.Rocket`.

✔ It matches the reference look; moving `Root.CFrame` moves the whole model; fin wedges face the right way.

## Phase 2 — Flight (the most important phase)

**2a.** Fixed-step simulation, render interpolation, input (`Branch.Keybinds`: WASD/stick rotation, Space/RT thrust, all three platforms), orbit camera.

**2b.** Contacts: collide-and-slide, crash, landing with the settle wobble, standing, takeoff blend.

**2c.** Effects: flame, trail, ignition, jet wash, landing dust, sparks; engine, wind and jet-wash sounds.

**2d.** Tuning on the flight gym with live `FlightTuning` attributes and the debug view. Compare the gravity presets ([13](13-reference-physics.md#gravity)) first; playtests on PC, phone and gamepad.

✔ All target metrics in [05](05-flight.md#tuning) are met. Flying is fun with no goal at all. Fuel exists in code behind `fuelEnabled` but is off.

## Phase 3 — Server authority and replication

- `Launch` / `RunStarted` / `RunEnded`, character parking, `State` stream, validation ([11-math §15](11-math.md)), `Correct`, `Snapshot` batching, remote interpolation, run end conditions (timeout, bounds).

✔ Clients and Servers test with 2–3 players and 150–200 ms lag: other rockets fly smoothly; teleport and speed hacks are corrected and end in `Rejected`; a lag spike alone never ends a run.

## Phase 4 — Guards

- `GuardService` AI states, vision, per-rocket suspicion, firing; `GuardSnapshot`; `GuardController` turret interpolation, vision cone, alert icons; HUD suspicion meter.

✔ Moving in view gets you shot in about 0.7 s; standing disguised in view survives for about 4 s at Disguise 0; the "turret turns straight at you" moment happens and reads clearly.

## Phase 5 — Targets, rewards, progression

- `TargetService` (HP, regen, damage shares, respawn), explosions (crash, contact, detonate), kill cam.
- Data template ([04](04-data-progression.md)), cash, stats, `ProgressionService` (XP, levels, `LevelUp`), stealth bonus.

✔ A kill pays the right cash and XP (checked against [11-math §13](11-math.md)); data survives a rejoin; two players sharing a kill both get paid by damage share.

## Phase 6 — Shop and meta

- `ShopService` + `ShopController`: four upgrades, skins (buy and equip), rebirth; level-gated Fuel upgrade.

✔ Every purchase is validated on the server; forged or spammed requests change nothing; upgrades change the flight and blast numbers exactly as specified.

## Phase 7 — PRO servers

- PRO place in the experience plus a Dev experience; `ModeService` (server mode, gate on join, teleports with retries, teleport GUI); PRO zones; PRO multipliers through `RunConfig`; PRO tag; PRO leaderboard; the release checklist in use ([08](08-pro-servers.md)).

✔ In the Dev experience: Main → PRO → Main keeps all data; a player below level 10 cannot get into PRO by any route; a failed teleport shows a clear message.

## Phase 8 — Zone 1 art pass and polish

- Replace the greybox with the rocky asset pack site by site; collision fidelity and perch checks; lighting, ambience, sound pass; mobile UI pass; performance pass on a mid-range phone at Low and High quality.

✔ Steady frame rate on a mid-range phone; no "invisible wall" or "flew through a rock" reports in a 10-player playtest.

## After phase 8

Everything in [backlog.md](backlog.md), monetization and the tutorial first.
