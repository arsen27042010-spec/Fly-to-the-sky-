# 07 — Guards, targets, explosions, rewards

Math: [11-math.md](11-math.md) §12–13, 16.

## Guard types

| Type | Is a target? | Example | Notes |
|---|---|---|---|
| **Sentinel** | Yes | Tank with a rotating turret (the reference) | Watches and must be hit: you strike when it looks away |
| **Watcher** | No | Turret on a ledge, searchlight tower | Protects approaches; cannot be destroyed in zone 1 |

A guard is a model tagged `Guard` with attributes `Kind` (`Sentinel` / `Watcher`), `Profile` (a key in `Config.GuardProfiles`), and an `Eye` attachment (the vision origin, which rotates with the turret).

## Guard profiles (Config)

| Profile | FOV | Range | Sweep amplitude | Peak sweep speed | Lock-on turn speed |
|---|---|---|---|---|---|
| `IndoorTank` | 70° | 45 | ±60° | 30°/s | 90°/s |
| `HeavyTank` | 80° | 55 | ±70° | 40°/s | 110°/s |
| `LedgeTurret` | 60° | 70 | ±45° | 25°/s | 90°/s |
| `Searchlight` | 40° | 90 | ±80° | 35°/s | 70°/s |

PRO servers add +15° FOV to every profile and ×1.5 suspicion gain (`RunConfig.suspicionMul`).

## AI states (server, GuardService, 10 Hz)

| State | Behaviour | Exit |
|---|---|---|
| `Sweep` | Sine sweep around the base yaw (§12.1) | Any rocket's suspicion > 0.35 ⇒ `Track` |
| `Track` | Turret turns toward the most suspicious visible rocket at the lock-on speed (§12.2) | Suspicion reaches 1 ⇒ `Fire`; all suspicion < 0.1 for 2 s ⇒ `Sweep` |
| `Fire` | Short wind-up (0.4 s, red cone, "!"), shot: `GuardFired`, the run ends with `Spotted` | ⇒ `Cooldown` |
| `Cooldown` | 2 s pause, then back to sweeping | ⇒ `Sweep` |

The server does **not** move turret parts. It computes yaw mathematically and sends `GuardSnapshot` at 12 Hz; clients rotate the turret locally with interpolation. Clients also draw the vision cone from the replicated yaw and alert level.

## Vision and suspicion

- **Vision test:** range, then cone, then line of sight against `Zones.*.Geometry`. The rocket's centre and nose are tested; either counts (§12.3).
- **Moving test:** throttle > 0.05, or server-measured speed over a 0.2 s window > 2 studs/s (§12.4).
- **Suspicion** is tracked per guard × rocket (§12.5):
  - visible and moving: about 0.7 s to get caught;
  - visible and still (disguised): about 4 s at Disguise 0, about 20 s at Disguise 20;
  - farther away is slower; out of view it decays at 0.5/s.
- The owner sees the highest suspicion across all guards on the HUD meter (`Status`, 10 Hz).
- **Latency fairness:** suspicion builds gradually, and "moving" uses a 0.2 s window, so a rocket that stopped a moment ago on the player's screen is not punished by a stale sample.

## Targets

A target is a model tagged `Target`, with attributes `Tier` (`Tank`, `HeavyTank`, `Bunker`, `SummitRadar`) and an invisible `Hitbox` part (an oriented box).

| Tier | HP | Base cash | Base XP | One-shot from Warhead level |
|---|---|---|---|---|
| Tank | 100 | 50 | 10 | 0 |
| Heavy tank | 250 | 250 | 25 | 9 |
| Bunker | 600 | 1,200 | 60 | 16 |
| Summit radar (PRO) | 1,500 | 6,000 | 150 | 24 |

- Damage persists; after 10 s without damage the target regenerates 5% of max HP per second.
- On death: a destruction effect, the target disappears, and it respawns after 20 s at full HP.
- Rewards are split between everyone who damaged it in the last 60 s, by damage share (§13.4). Weaker players can team up on tough targets, or chip away alone over several runs.

## Explosions

Every run ends in at most one explosion, which is the only way to deal damage:
- **Crash:** a hard, nose-first hit ([05](05-flight.md#surfaces));
- **Contact:** the rocket's path touches a target hitbox at any speed (§16.1);
- **Detonate:** the player presses detonate anywhere.

Damage to each target in the blast radius: `W · saturate(1 − d/Rb)`, with `d` measured to the hitbox surface (§13.1–13.2, §16.2). So sneaking right up to a target and detonating is a valid strike.

## Run end conditions

| Reason | Cause | Explosion? |
|---|---|---|
| `Detonated` | Player pressed detonate, or touched a target | Yes |
| `Crashed` | Hard nose-first impact | Yes |
| `Spotted` | A guard's suspicion reached 1 and it fired | No (shot down) |
| `Timeout` | 180 s Normal / 120 s PRO | No |
| `OutOfBounds` | Left the zone volume for 3 s (with a warning) | No |
| `Rejected` | Validation bucket full ([11-math §15.5](11-math.md)) | No |

`RunEnded` carries cash, XP, kills and the stealth flag. A stealth strike (peak suspicion < 0.5 for the whole run) pays ×1.5 and shows a "Silent strike" banner.

## Level design rules for guards

- Every Sentinel needs at least one perch spot within 15 studs that sits outside its cone for at least 40% of the sweep cycle.
- Every approach needs cover: a route where the rocket can stop out of view at least every 20 studs.
- Watchers never overlap so completely that a site has no gaps at all.
