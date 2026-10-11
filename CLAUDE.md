# Rules for the implementer

This repository is the design for **Fly to the Sky**. The game itself is built in Roblox Studio with the Branch framework. Whoever implements it (a person or an agent with access to the developer's machine) follows these rules.

## Before writing code

1. Read `README.md`, then `docs/` in numeric order. `docs/11-math.md` is the source of truth for formulas and default values.
2. Read `reference/Branch_API_Reference.md`. Check the installed runtime with `Branch.Version`; if it differs from v2.5.2, trust the installed source and the official docs over the reference file.
3. Inspect the existing Studio project before changing it: the Branch runtime, both `Main` scripts, existing Services/Controllers, the data template and Network declarations.
4. Work phase by phase as written in `docs/12-roadmap.md`. Do not start a phase before the previous one meets its acceptance criteria. Never pull in items from `docs/backlog.md` (monetization, tutorial and so on).

## Branch conventions

- Server logic goes in `…Service` ModuleScripts under `ServerScriptService.Services`, client logic in `…Controller` ModuleScripts under `StarterPlayerScripts.Controllers`. Helper modules must **not** end in `Service` or `Controller`.
- Resolve other segments with `Branch:GetSegment` in `Init`/`Start` or later, never at a module's top level. Declare `Dependencies`.
- Networking only through literal `Service.Network` tables (no variables, loops or calls inside), with typed, bounded payloads and `Rate`/`Cooldown` on every client-originated entry. No plain RemoteEvents.
- After editing scripts with tools (MCP, Rojo, `UpdateSourceAsync`), press **Compile** in Branch Studio, or send the `compile` command, then check Problems. Compile in Edit mode, never in Play mode.
- Persistent data only through `Branch.Data.Server` writes (`Set`, `Update`, `Increment`, `Insert`, `Remove`). Wait for loading (`Data.Await`, `Data.IsLoaded`). Keep `Mock = RunService:IsStudio()`.
- Write `--!strict` Luau with types. Clean up connections with `Util/Scythe`.

## Game rules that must never be broken

- The server is authoritative over outcomes: hits, damage, rewards, XP, purchases, fuel, level gates. The client only simulates its own rocket and asks.
- Validate every client request: data loaded, state allows it, cooldowns, server-measured distance and speed, prices from server config only.
- Flight constants live as attributes on `ReplicatedStorage.Shared.FlightTuning` so they can be tuned live; defaults come from `docs/11-math.md`.
- No `Instance.new` in per-frame effect code: pool everything.
- Keep the Main and PRO places on the same build (see the release checklist in `docs/08-pro-servers.md`).

## Testing

- Every phase: Studio **Test › Clients and Servers** with 2+ players and incoming replication lag of 150–200 ms. Read Output on both sides.
- Check data loading and saving, rejoining, invalid and forged requests, and a run ending in every possible way.
- Teleports do not work in Studio: test PRO flows in the Dev experience.
- Never claim something works without having run it.

## Reporting

- For every file you create or change, state the exact Studio path and its type (Script, LocalScript, ModuleScript).
- When a design decision changes during implementation, update the matching document in `docs/` in the same change. The docs must keep describing the real game.
- Documentation and code comments are in English.
