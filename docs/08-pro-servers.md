# 08 — PRO servers

PRO is a separate place in the same experience, open from **player level 10**. Fuel exists only there.

## Normal vs PRO

| | Normal | PRO |
|---|---|---|
| Fuel | None | Refills only while standing ([11-math §3.3](11-math.md)) |
| Guards | Base profiles | +15° FOV, suspicion ×1.5 |
| Timeout | 180 s | 120 s |
| Rewards | ×1 | Cash ×2, XP ×1.5 |
| Content | Zone sites | Plus PRO sites (`ServerStorage.ProZones`) |
| Prestige | — | PRO tag above the rocket, PRO leaderboard, milestone skins (backlog) |

## Experience layout

- One experience, two places: **Main** (the start place) and **PRO**.
- **One place file, published to both place ids.** At startup `ModeService` decides the mode with `game.PlaceId == Config.PlaceIds.Pro`. For Studio tests, the attribute `workspace.ForceServerMode = "Pro"` overrides it.
- Why not Roblox Packages: Branch Studio writes generated code into `Packages.Branch` and `ServerScriptService.BranchNetworkServer`. A single file guarantees identical code, networking and data template in both places.
- **A separate Dev experience** (its own Main and PRO) is required to test teleports: they do not work in Studio, and the live game is not a test bench.

## What the PRO server does differently

- Every run uses `RunConfig.mode = "Pro"`: fuel on, PRO guard modifiers, PRO multipliers.
- `ZoneService` moves `ServerStorage.ProZones` into `Workspace.Zones` at startup; Main servers never load them.
- The hub uses PRO dressing and has a "Back to Normal" portal.
- Player limits: Main 16–20, PRO about 12. Roblox fills public servers by itself, so low PRO population still gathers players together.

## Teleports

**Main → PRO**
1. The player uses the PRO portal; the client calls `ModeService.JoinPro`.
2. The server checks: data loaded, `level ≥ 10`, not in a run, not already teleporting.
3. `TeleportService:TeleportAsync(Config.PlaceIds.Pro, { player }, options)`, with `TeleportData = { buildVersion = Config.BUILD_VERSION }`.
   - Wrap it in `pcall` and retry up to 3 times with growing delays.
   - Handle `TeleportService.TeleportInitFailed`: retry, then report `TeleportStatus.Failed` to the player.
4. Branded transition: `TeleportService:SetTeleportGui` ("Flying to PRO…"). The arriving place picks it up in `ReplicatedFirst` with `GetArrivingTeleportGui()` and keeps it until the data has loaded.

**Gate on arrival, always.** It applies even when the player came through "join friend" or a direct link:
1. `Data.Await(player)`.
2. `level < 10` ⇒ message "PRO opens at level 10" and teleport back to Main.
3. If that teleport fails ⇒ kick with the same message.

**PRO → Main:** the portal in the PRO hub calls `ModeService.ReturnToNormal`.

**Data across places:** the session lock is handled by ProfileStore inside Branch Data. The old server saves and releases the profile; the new one waits. Verify in the Dev experience that nothing is lost, including fast back-and-forth hops.

**Build check:** the arriving server compares `TeleportData.buildVersion` with its own and logs a mismatch to analytics. A mismatch means only one place was published.

## Motivation inside Main

- The Main hub shows the PRO leaderboard (an OrderedDataStore, shared by the experience) and the PRO portal with a "LVL 10" lock and a progress label ("Level 7/10").
- On reaching level 10:
  - a "PRO unlocked!" celebration;
  - a short fuel lesson on the practice course;
  - the portal lights up.

## Release checklist

Mandatory, because one file goes into two places:
1. Bump `BUILD_VERSION` in `Shared/Config`.
2. Branch Studio: Compile; Problems must be empty.
3. Publish to **Main**.
4. Publish to **PRO** (File › Publish to Roblox As › the PRO place).
5. Creator Hub: **Migrate to Latest Update** for **both** places.
6. In the live game: teleport Main → PRO → Main and check the data.

**Never update only one place.** A server running an older data template could misread a save written by a newer one. Later, publishing to both places can be automated through Open Cloud ([backlog](backlog.md)).
