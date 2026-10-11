# 03 — Network

All entries are Branch compiled networking: literal `Service.Network` tables, generated endpoints `Branch.Network.Server.<Service>.<Event>` / `Branch.Network.Client.<Service>.<Event>`. Every client-originated entry has a `Rate`; the server validates gameplay on top of Branch's packet checks.

Entries with `Return` are functions (client invokes, server answers). Otherwise `From` says who sends.

## RocketService

| Entry | From | Type | Data | Limits / notes |
|---|---|---|---|---|
| `Launch` | function | Reliable | `"u8"` zoneId → `Return = "enum(Ok, NotLoaded, InRun, Locked, Cooldown)"` | `Rate = "1/s"`, `Cooldown = 2` |
| `State` | Client | **Unreliable** | `{ t = "f64", cf = "CFrame(compact)", vel = "vector(f16)", throttle = "f32(0..1, 0.05)" }` | `Rate = "40/s"`; client sends 30 Hz |
| `Detonate` | Client | Reliable | — | `Rate = "2/s"`; explode at the current position |
| `RunStarted` | Server | Reliable | `{ runId = "u16", zoneId = "u8", spawn = "CFrame", fuelEnabled = "boolean", fuel = "f32(0..20, 0.05)", timeout = "u16" }` | |
| `RunEnded` | Server | Reliable | `{ runId = "u16", reason = "enum(Detonated, Crashed, Spotted, Timeout, OutOfBounds, Rejected)", cash = "u32", xp = "u16", stealth = "boolean", kills = "u8" }` | |
| `Correct` | Server | Reliable | `{ cf = "CFrame", vel = "vector(f16)" }` | Server rejected a move; the client snaps back |
| `RocketAdded` | Server | Reliable | `{ id = "u16", player = "Instance(Player)", skin = "string(..24)" }` | To everyone |
| `RocketRemoved` | Server | Reliable | `{ id = "u16", reason = "enum(Detonated, Crashed, Spotted, Timeout, OutOfBounds, Rejected)", at = "vector" }` | Others play the matching effect |
| `Snapshot` | Server | **Unreliable** | `"{ id: u16, t: f64, cf: CFrame(compact), vel: vector(f16), throttle: f32(0..1, 0.05) }[..64]"` | One batch of all active rockets, 20 Hz |
| `Status` | Server | Unreliable | `{ suspicion = "f32(0..1, 0.01)", fuel = "f32(0..20, 0.05)?" }` | `Hz = 10`; to the rocket's owner only; `fuel` only on PRO |

## GuardService

| Entry | From | Type | Data | Notes |
|---|---|---|---|---|
| `GuardSnapshot` | Server | Unreliable | `"{ id: u8, yaw: f32(-180..180, 0.5), alert: f32(0..1, 0.01) }[..64]"` | One batch, 12 Hz; clients interpolate turret yaw |
| `GuardFired` | Server | Reliable | `{ id = "u8", at = "vector" }` | Shot effect and sound |

## TargetService

| Entry | From | Type | Data | Notes |
|---|---|---|---|---|
| `TargetHealth` | Server | Reliable | `{ id = "u16", hp = "f32(0..1, 0.01)" }` | On change, for health bars |
| `TargetDestroyed` | Server | Reliable | `"u16"` | |
| `TargetRespawned` | Server | Reliable | `"u16"` | |

## ProgressionService

| Entry | From | Type | Data | Notes |
|---|---|---|---|---|
| `LevelUp` | Server | Reliable | `{ level = "u16", unlocked = "string(..32)[..8]" }` | Celebration and unlock list; XP and level themselves are read from `Data.Client` |

## ShopService

| Entry | From | Type | Data | Limits |
|---|---|---|---|---|
| `BuyUpgrade` | function | Reliable | `"enum(speed, disguise, warhead, fuel)"` → `Return = "boolean"` | `Rate = "5/s"`; `fuel` requires level ≥ 10 |
| `BuySkin` | function | Reliable | `"string(..24)"` → `Return = "boolean"` | `Rate = "3/s"` |
| `EquipSkin` | function | Reliable | `"string(..24)"` → `Return = "boolean"` | `Rate = "3/s"` |
| `Rebirth` | function | Reliable | — → `Return = "boolean"` | `Rate = "1/s"`, `Cooldown = 2` |

## ModeService

| Entry | From | Type | Data | Limits |
|---|---|---|---|---|
| `JoinPro` | function | Reliable | — → `Return = "enum(Ok, LowLevel, InRun, Busy, Failed)"` | `Rate = "1/s"`, `Cooldown = 5` |
| `ReturnToNormal` | function | Reliable | — → `Return = "enum(Ok, InRun, Busy, Failed)"` | `Rate = "1/s"`, `Cooldown = 5` |
| `TeleportStatus` | Server | Reliable | `"enum(Teleporting, Retrying, Failed)"` | |

## Rules and notes

- **Batch multi-entity streams yourself.** `Snapshot` and `GuardSnapshot` carry every entity in one packet on a fixed tick. Do not use `Hz` or `Latest` on a per-entity event: as documented, they keep only the newest value per recipient and would drop other entities' updates. Verify this against the Branch docs before relying on it either way.
- **Timestamps:** clients send `t = workspace:GetServerTimeNow()`. The server clamps it ([11-math §15.8](11-math.md)) and forwards it in `Snapshot`, so remote interpolation follows the sender's real timing.
- **Enum names** are plain identifiers; mirror them in `Shared/Config` so server and client code share them.
- **`OnReject`:** log refused packets in Studio; never punish a single rejection (lag and bugs cause them too).
- **Anticheat:** leave the ready-made movement checks off for characters (they are parked during runs). Rocket movement is validated in `RocketService` ([11-math §15](11-math.md)).
