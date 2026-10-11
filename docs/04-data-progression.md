# 04 — Data and progression

## Data template

Started once in `PlayerService:Init()`:
```lua
Branch.Data.Server.Start({
	Template = {
		cash = 0,
		totalCash = 0,        -- lifetime earnings, never decreases (leaderboards, analytics)
		level = 1,
		xp = 0,               -- progress inside the current level
		rebirths = 0,
		upgrades = { speed = 0, disguise = 0, warhead = 0, fuel = 0 },
		skins = { "Flower" }, -- owned skin ids
		equippedSkin = "Flower",
		stats = { launches = 0, strikes = 0, stealthStrikes = 0, spotted = 0, crashes = 0 },
		pro = { launches = 0, strikes = 0, bestStreak = 0 },
		settings = { sensitivity = 1, invertY = false, quality = "High" },
	},
	Store = "PlayerData",
	Mock = RunService:IsStudio(),
	Leaderstats = { level = "Level", cash = "Cash" },
	Migrations = {},
})
```
- One experience means one DataStore: the Main and PRO places share this data ([08](08-pro-servers.md)).
- New fields reach old saves automatically. Removing a field or changing a type needs a migration.
- Game-pass ownership is not stored; it is checked on join (when monetization arrives, [backlog](backlog.md)).
- Monetization adds a `purchases` field later (processed receipt ids). Adding fields is safe.

## Field ownership

Exactly one service writes each field. Everyone else reads.

| Field | Writer |
|---|---|
| `cash`, `totalCash` | TargetService (rewards), ShopService (spending) — both through `Formulas` and `Data.Update`, never yielding between check and write |
| `level`, `xp` | ProgressionService (TargetService calls `ProgressionService:AddXp`) |
| `rebirths`, `upgrades` | ShopService |
| `skins`, `equippedSkin` | ShopService |
| `stats.*`, `pro.*` | RocketService (launches, crashes, spotted), TargetService (strikes) |
| `settings.*` | PlayerService (through a settings request, added with the settings UI) |

## Player level and XP

- Formula: `need(L) = round(25 · L^1.35)`. From 1 to 10 takes about 2,100 XP, roughly 45–60 minutes ([11-math §14](11-math.md)).
- XP per kill by target tier, ×1.5 for stealth, ×1.5 on PRO, split by damage share. +2 XP for every finished run, even a failed one: younger players keep growing while they learn.
- Rebirth does **not** reset level or XP.

**What levels unlock**

| Level | Unlock |
|---|---|
| 1 | Zone 1, sites A–C (tougher targets are gated by Warhead, not by level) |
| 10 | PRO servers; the Fuel upgrade appears in the shop |
| later | Further zones (each new zone gets its own level gate) |

On level-up: `LevelUp` event, a celebration effect, and a toast listing what unlocked.

## Upgrades

| Upgrade | Effect | Base cost | Cap |
|---|---|---|---|
| Speed | `vmax = 60 + 3·L` studs/s | 25 | 15 |
| Disguise | still-in-view suspicion × `(1 − 0.04·L)` | 40 | 20 |
| Warhead | damage `100 · 1.12^L`, blast radius `6 + 0.1·L` | 50 | 30 |
| Fuel (level 10+) | `fuelMax = 3 + 0.5·L` seconds; only matters on PRO | 30 | 14 |

Cost of the next level: `round(base · 1.35^L)`. The Warhead is the content gate: the Heavy tank needs level 9 for a one-shot kill, the Bunker 16, the PRO radar 24.

## Rebirth

- Cost: `10,000 · 2.5^R` cash.
- Resets `cash` and `upgrades`. Keeps level, XP, skins and stats.
- Permanent cash multiplier: `1 + 0.5·R`.

## Skins

- A skin = hull colours + a disguise part set (`Assets/Disguises/<SkinId>`). The first skin is `Flower` (flower vase), the one from the reference.
- Bought with cash in the shop; prices live in `Shared/Skins`.
- Disguise effectiveness is the same for every skin for now. A context bonus (vase skin near tables and flower pots) is in the [backlog](backlog.md).

## Safety rules

- Prices, rewards and XP are computed only on the server from `Shared/Config` and `Formulas`.
- Purchases check and write without yielding in between ([Branch reference §13](../reference/Branch_API_Reference.md)).
- Rewards are paid once per kill and per contributor, keyed by `runId`.
