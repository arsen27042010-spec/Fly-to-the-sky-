# Branch Framework API Reference for Claude
**Target:** Roblox Luau / Branch Framework **v2.5.2** (runtime and Branch Studio plugin)  
**Purpose:** A practical, Claude-ready reference for building a Roblox game with Branch.  
**Primary source:** the official Branch documentation and the Branch 2.5.2 runtime source.

- Official documentation: https://praxprix-studios.github.io/Branch-Framework/website/index.html
- Getting started: https://praxprix-studios.github.io/Branch-Framework/website/getting-started.html
- Segments: https://praxprix-studios.github.io/Branch-Framework/website/guide/segments.html
- Player data: https://praxprix-studios.github.io/Branch-Framework/website/guide/data.html
- Networking: https://praxprix-studios.github.io/Branch-Framework/website/guide/network.html
- Branch Studio (the plugin): https://praxprix-studios.github.io/Branch-Framework/website/guide/studio.html
- API reference: https://praxprix-studios.github.io/Branch-Framework/website/api/branch.html

> **Version note:** This document targets Branch v2.5.2. Check the installed runtime with `Branch.Version` (or the `BranchVersion` attribute on `ReplicatedStorage.Packages.Branch`). If it is older, Branch Studio's Overview page shows an **Update** button. If the installed runtime or generated API differs from this document, trust the project and the official docs over this file. Branch Studio generates parts of the API (networking, types) from your code.

---

## 1. Instructions for Claude (read first)

When implementing or modifying this game:

1. **Use Branch as the game's framework.** Server logic goes into `...Service` ModuleScripts, client logic into `...Controller` ModuleScripts, unless the project's Segment Rule says otherwise (see §3).
2. **Inspect the existing Roblox project before changing it.** Find the installed Branch runtime, the `Main` scripts, Services, Controllers, the data template and existing `Network` declarations. Keep the project's structure and conventions.
3. **Do not invent Branch methods.** Use only APIs described here or confirmed in the official docs or the installed source. If a needed API is unclear, inspect the project and the docs before coding.
4. **Keep authoritative gameplay on the server.** The client asks; the server validates the request, ownership, distance, cooldown, state, costs, rewards and permissions before applying it.
5. **Use `Branch.Data.Server` for persistent player data.** Declare every persistent field in the template. Write only through `Set`, `Update`, `Increment`, `Insert` and `Remove`; changing a table returned by `Get` neither saves nor replicates.
6. **Use Branch networking declarations.** Put a literal `Service.Network` table in the service that owns the events. Do not create plain RemoteEvents for things Branch networking covers.
7. **Keep `Service.Network` a literal table.** Branch Studio reads it from the source text: no variables, loops or function calls inside it.
8. **Compiled networking needs Branch Studio.** Its **Network** feature must be on (Overview page, or Settings › Project › Compiled networking), and the declarations must have compiled before `Branch.Network` exists. See §12 for compiling when you edit scripts from tools.
9. **Use typed, bounded payloads.** Choose the smallest suitable type, add ranges and length limits, and add `Rate` and guards (`Cooldown`, `Alive`, `Near`, `Owned`) to client-originated actions. Never trust client-sent prices, damage, rewards, inventory ownership or progression.
10. **Respect data loading.** Reads return `nil` until the profile loads, and writes before that error. Use `Data.Await(player)`, `Data.OnLoaded(...)` or `Data.IsLoaded(player)`.
11. **Preserve old saves.** New template fields reach old saves automatically; removing fields or changing types needs a migration. Do not casually change a live template's shape.
12. **Resolve other segments in `Init`, `Start` or later**, never at a module's top level.
13. **Clean up connections and tasks.** `Util/Scythe` (cleanup scopes) is included for that.
14. **Write `--!strict` code with types.** Branch's API is fully typed under both the old and the new Luau type solver (§11). Annotate a segment method's return type whenever it returns something the plugin cannot work out from the code.
15. **Test the actual flows.** Check server and client Output, data loading, saving, multiplayer, respawns, invalid requests and UI state. Do not claim something works without testing it.
16. **When returning code, state exact paths and whether each file is a Script, LocalScript or ModuleScript.** If asked for full replacement scripts, give complete files, not fragments.

---

## 2. Installation and startup

The runtime lives at `ReplicatedStorage.Packages.Branch`. Install it with Branch Studio (recommended: Overview › Install), pesde, the `Branch.rbxm` model, or Rojo, as the Getting started page describes. Branch Studio matters most: it compiles the networking declarations and the types.

### Default folder layout

```text
ReplicatedStorage
└─ Packages
   └─ Branch                 (ModuleScript: the runtime)

ServerScriptService
├─ Main                      (Script)
└─ Services                  (Folder)
   ├─ PlayerService          (ModuleScript)
   └─ ExampleService         (ModuleScript)

StarterPlayer
└─ StarterPlayerScripts
   ├─ Main                   (LocalScript)
   └─ Controllers            (Folder)
      └─ ExampleController   (ModuleScript)
```

These are Branch Studio's default roots (Settings › Project › Paths); a project may use others.

### Server entry point

`ServerScriptService/Main` (Script):

```lua
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

Branch:Start(script.Parent.Services)
```

### Client entry point

`StarterPlayer/StarterPlayerScripts/Main` (LocalScript):

```lua
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

Branch:Start(script.Parent:WaitForChild("Controllers"))
```

Call `Branch:Start(...)` once on each side: the server starts the services, the client the controllers.

---

## 3. Core `Branch` API

```lua
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)
```

### `Branch:Start`

```lua
Branch:Start(...roots: Instance, options: { SegmentRule: "Suffix" | "Folders" }?)
```

Finds the segments under one or more roots, requires them, runs every `Init` in dependency order, then every `Start` (each in its own thread), then connects the hooks.

- Rule `"Suffix"` (default): every ModuleScript under the roots whose name ends with `Service` (server) or `Controller` (client), at any depth and in any casing, is a segment; other modules are helpers.
- Rule `"Folders"`: modules directly under a root, or under Folders inside it, are segments; a ModuleScript inside another ModuleScript is that module's helper.
- The rule comes from the `options` table, else from the `SegmentRule` attribute Branch Studio puts on the Branch module, else `"Suffix"`.
- Errors: calling `Start` twice; passing roots in a table (`Branch:Start({ a, b })`) instead of as separate arguments; an unknown option; a non-Instance root.

```lua
Branch:Start(folderA, folderB, { SegmentRule = "Folders" })
```

### Members

| Member | Meaning |
|---|---|
| `Branch.Version` | Runtime version string, `"2.5.2"`. |
| `Branch:GetSegment(name)` | The segment with that name; errors if there is none yet. Typed per segment (§11). Use in `Init` or later. |
| `Branch:LocateSegments({ "ShopService", "CoinService" })` | A map name → segment; errors for an unknown name. |
| `Branch:AwaitStart()` | Yields until every segment has been initialised and started. |
| `Branch:GetPhase()` | `"Idle"`, `"Loading"`, `"Initializing"` or `"Started"`. |
| `Branch:GetReport()` | `{ { Name, Path, Status, InitMs } }`; `Status` is `"Loaded"`, `"Initialized"`, `"Failed"` or `"Started"`. |
| `Branch.Data.Server` / `Branch.Data.Client` | Player data (§5, §6). Each errors when used on the wrong side. |
| `Branch.Network.Server` / `.Client` | Compiled networking (§7). `Branch.Network` errors until Branch Studio has compiled it. |
| `Branch.Network.Group(name)` | Network groups (server). |
| `Branch.Network.OnReject(handler)` | Refused client packets (server). |
| `Branch.Network.Anticheat` | The anticheat (§9). |
| `Branch.Keybinds` | The Keybinds folder Branch Studio's Keybinds page edits: `Branch.Keybinds.<Context>.<Action>` are Roblox `InputContext` / `InputAction` instances, typed. Errors until a context exists. |

A typo such as `Branch.Netwrok` errors on purpose. Utilities are ModuleScripts inside `Packages.Branch.Util`, not fields of `Branch` (§10).

---

## 4. Segments: services, controllers, lifecycle, hooks

A segment is a ModuleScript that returns a table. Under the default rule: `PlayerService`, `ShopService`, `CombatService` on the server; `HudController`, `ShopController`, `CombatController` on the client. Other modules are helpers: they are not started, you require them from a segment.

### Lifecycle

1. **Load:** Branch requires each segment. A module that errors or does not return a table is reported and skipped. A second segment with the same name is reported and skipped.
2. **Init:** `Init` runs one segment at a time, in dependency order. It may yield; later Inits wait (a warning appears after 5 seconds). A failing Init marks that segment `"Failed"`.
3. **Start:** after every Init, each segment's `Start` runs in its own thread.
4. **Hooks:** connected once per hook type, shared by every segment that has it.

### Dependencies

```lua
local ShopService = {
	Dependencies = { "PlayerService" }, -- PlayerService:Init runs before ShopService:Init
}

function ShopService:Init() end
function ShopService:Start() end

return ShopService
```

Avoid dependency cycles (they are reported) and duplicate names.

### Hooks

| Hook | When |
|---|---|
| `PlayerAdded(player)` | For every player already in the game when Branch starts, then for every new player. Each call in its own thread. |
| `PlayerRemoving(player)` | When a player leaves. On the server their data is still there: writes made before the hook yields are saved with the session. |
| `Heartbeat(dt)` | Every frame, after physics. |
| `PreSimulation(dt)` | Every frame, before physics. |
| `PreRender(dt)` | Every frame, before rendering. Client only. |

A hook that errors is reported (at most once every 5 seconds per segment) and does not stop the other segments' hooks.

```lua
function ExampleService:PlayerAdded(player: Player) end
function ExampleService:PlayerRemoving(player: Player) end
function ExampleController:PreRender(dt: number) end
```

### Finding another segment

```lua
local ShopService = {}
local PlayerService -- set in Init

function ShopService:Init()
	PlayerService = Branch:GetSegment("PlayerService")
end
```

Never call `Branch:GetSegment` at a module's top level: the other segment may not be loaded yet.

---

## 5. Player data: server

```lua
local Data = Branch.Data.Server
```

Branch keeps player data in ProfileStore (session-locked) and mirrors each player's data to their own client. Every write is checked against the template, nested fields included; NaN and infinity are refused.

### Start data once

From an early server segment, such as `PlayerService:Init()`:

```lua
local RunService = game:GetService("RunService")
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local PlayerService = {}

function PlayerService:Init()
	Branch.Data.Server.Start({
		Template = {
			coins = 0,
			stats = { kills = 0, best = 0 },
			items = {},
		},
		Store = "PlayerData",
		Mock = RunService:IsStudio(),
		Leaderstats = { coins = "Coins" },
		Migrations = {},
	})
end

return PlayerService
```

`Mock = RunService:IsStudio()` keeps Studio tests in memory, away from real saves. Do not remove it without deciding on purpose how test data should behave.

### `Data.Start(config)`

| Field | Meaning |
|---|---|
| `Template` | Required. The default data, its shape and its types. |
| `Store` | ProfileStore name; default `"PlayerData"`. |
| `KeyPrefix` | DataStore key prefix; default `"PLAYER_"`. |
| `Mock` | Keep data in memory only (tests). |
| `DontSave` | Load real data but never write it. |
| `Migrations` | Array of functions: `Migrations[n]` upgrades version n-1 to n, on a copy. |
| `Leaderstats` | Array of data paths (shown name = the last part) or a map path → shown name. |

Calling `Start` twice, or without a `Template` table, errors.

### Data paths

A dot-separated string, or an array of keys:

- `"coins"`, `"stats.kills"`, `"items.3.id"`, `"friends.12345"`
- `{ "stats", "kills" }`

Numeric parts index arrays; in dictionaries they address string keys. In an empty table, index `1` starts an array and any other number starts a dictionary (DataStores cannot save mixed keys). With Branch Studio's Data types on, paths autocomplete from the template.

### Server data functions

| Function | Behavior |
|---|---|
| `Data.Get(player, path?)` | The value, or the whole data table; `nil` until loaded. Never errors. |
| `Data.Set(player, path, value)` | Checked write, synced to the client. |
| `Data.Update(player, path, transform)` | `transform(old)` returns the new value; checked, written, returned. |
| `Data.Increment(player, path, delta?)` | Adds `delta` (default 1, may be negative); returns the new number. |
| `Data.Insert(player, path, value, index?)` | Inserts into an array (at the end by default). |
| `Data.Remove(player, path, index)` | Removes and returns an array element. |
| `Data.Observe(player, path, callback)` | `callback(new, old)` once loaded, then on every change (also when a parent table is replaced); returns a disconnect function. |
| `Data.Await(player)` | Yields until loaded; returns the data, or `nil` if the player left first. |
| `Data.IsLoaded(player)` | Whether the data is loaded. |
| `Data.OnLoaded(callback)` | `callback(player, data)` for players already loaded, then for each new load (never twice for one load); returns a disconnect function. |
| `Data.Peek(userId)` | Yields; reads a user's saved data without taking their session lock. |
| `Data.Reset(player)` | Wipes the profile and kicks the player; the next join starts from the template. |
| `Data.Send(userId, key, payload)` | A message to a player's profile, delivered even across servers; returns whether it was sent. |
| `Data.OnMessage(key, handler)` | One handler per key: `handler(player, payload)`; return `false` to keep the message for later, anything else marks it handled. |
| `Data.GetProfile(player)` | The raw ProfileStore profile or `nil`. Writing through it bypasses Branch's checks and syncing: avoid. |

```lua
local Data = Branch.Data.Server

local data = Data.Await(player)
if not data then
	return -- the player left before loading finished
end

Data.Increment(player, "coins", 10)
Data.Update(player, "stats.best", function(old)
	return math.max(old, newScore)
end)
Data.Insert(player, "items", { id = 7 })

local stop = Data.Observe(player, "coins", function(new, old)
	print("coins", old, "->", new)
end)
-- stop() when no longer needed
```

### Loading rules and data safety

- Before loading, `Get` returns `nil` and every write errors.
- Writes must fit the template all the way down; `Insert` checks new elements against the template array's example element when it has one.
- Prefer `Update` for read-modify-write.
- Do not change tables returned by `Get`.

### Migrations

- New template fields are added to old saves automatically.
- Removed fields and changed types are not: write a migration. Branch Studio warns under Problems when the template's shape changed in a way saves will not follow.
- Migrations run on a copy; if one fails, the saved data stays untouched and the player is asked to rejoin.

### Player leaving

`PlayerRemoving` hooks can still read and write the leaving player's data; do the writes **before yielding**.

---

## 6. Player data: client

```lua
local Data = Branch.Data.Client
```

A read-only mirror of the local player's data. It starts syncing when required; there is nothing to start. Only the server writes: ask it with a network event. The client never sees other players' data. If the server never calls `Data.Server.Start`, the client warns after 15 seconds.

| Function | Behavior |
|---|---|
| `Data.Get(path?)` | The value, or the whole table; `nil` until the data arrives. |
| `Data.IsLoaded()` | Whether the data has arrived. |
| `Data.Await()` | Yields until the data arrives and returns it. |
| `Data.OnLoaded(callback)` | `callback(data)` once: at once if loaded, else when it arrives. |
| `Data.Observe(path, callback)` | `callback(value, nil)` once loaded, then `callback(new, old)` on every change; returns a stop function. |
| `Data.OnInserted(path, callback)` | `callback(index, value)` when the server inserts into that array; returns a disconnect function. |
| `Data.OnRemoved(path, callback)` | `callback(index, value)` when the server removes from that array; returns a disconnect function. |

```lua
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local HudController = {}

function HudController:Start()
	local label: TextLabel = -- your TextLabel
	Branch.Data.Client.Observe("coins", function(coins)
		label.Text = `Coins: {coins}`
	end)
end

return HudController
```

---

## 7. Networking

### Requirements

1. Branch Studio's **Network** feature is on (Overview, or Settings › Project › Compiled networking).
2. Events and functions are declared in the `Network` table of the service that owns them.
3. Branch Studio has compiled them (it does after each pause in typing, on its toolbar's Compile button, and on the `compile` command, §12).
4. You use the generated endpoints:
   - server: `Branch.Network.Server.<ServiceName>.<Event>`
   - client: `Branch.Network.Client.<ServiceName>.<Event>`

The server half is generated into `ServerScriptService.BranchNetworkServer`, so players never download the server's checks. Using `Server` on a client or `Client` on the server is an error.

### Declarations

`Service.Network` must be a literal table:

```lua
local ShopService = {}

ShopService.Network = {
	Buy          = { Data = "string(..32)", Return = "boolean", Rate = "4/s" }, -- a function
	CoinsChanged = { From = "Server", Data = "u32" },
	Ready        = { From = "Client" },                                        -- no data
	Move         = { From = "Server", Unpack = true, Data = { pos = "vector", yaw = "f32" } },
}

return ShopService
```

A `Return` makes the entry a function: a client asks, the server answers; functions have no `From`.

### Options

| Option | Values | Default |
|---|---|---|
| `From` | `"Server"` or `"Client"`: who sends. Required for events. | |
| `Type` | `"Reliable"` or `"Unreliable"`. | `"Reliable"` |
| `Call` | `"Async"`: each listener call in a pooled thread. `"Sync"`: inline in the receive loop (fastest). | `"Async"` |
| `Data` | What travels (§8). | nothing |
| `Return` | What a function returns; makes the entry a function. | |
| `Unpack` | `true`: a struct travels as separate arguments, no table. | `false` |
| `Rate` | `"15/s"` or `"600/m"`: how many a client may send. | no limit |
| `Cooldown` | Seconds between a player's accepted requests (a fifth early is allowed, for lag). | |
| `Alive` | `true`: the player's Humanoid must have Health above 0. | |
| `Near` | `20`: the first Instance or vector in `Data` must be within 20 studs of the player's character; `{ at = 30 }` names the field. | |
| `Owned` | `true`: the first Instance in `Data` must be in the player's character or backpack; `"tool"` names the field. | |
| `Latest` | `true`: several sends to one recipient in a frame send only the last. | `false` |
| `Hz` | Up to 60: the newest value goes out at most this often per second. | every frame |
| `Lod` | Distance rings, near to far: `{ { 50, 30 }, { 200, 5 } }`; adds `FireFrom`. | |
| `Delta` | `true`: only the struct fields that changed travel. Reliable only. | `false` |
| `Key` | With `Delta`: the field that tells objects apart, such as `"id"`. | |
| `Threshold` | With `Delta`: how far a number or vector may move and still count as unchanged; one number or per field `{ pos = 0.1 }`. | |
| `Predict` | With `Delta`, server → client: `{ pos = 0.25 }`; the client guesses between updates (adds `At` on the client, `Exact` on the server). | |
| `Pack` | `true`: small numbers, enums and booleans share a bit stream. | `false` |

Guards (`Cooldown`, `Alive`, `Near`, `Owned`) and `Rate` apply only to client events and functions. `Near` and `Owned` cannot be combined with `Delta`. A typo in an option is a compile error that suggests the right name.

### The shape of `Data`

| You write | You send |
|---|---|
| nothing | `Fire()` |
| `Data = "u8"` | `Fire(5)` |
| `Data = { "u16", "boolean" }` | a tuple: `Fire(25, true)` |
| `Data = { pos = "vector", yaw = "f32" }` | a struct table: `Fire({ pos = p, yaw = 1.5 })` |
| the same with `Unpack = true` | the fields as arguments, in declared order: `Fire(p, 1.5)` |

### Server → client events (`From = "Server"`)

Server:

```lua
local Net = Branch.Network.Server.ShopService

Net.CoinsChanged.Fire(player, coins)               -- one player
Net.CoinsChanged.FireAll(coins)                    -- everyone (sent once)
Net.CoinsChanged.FireList({ a, b }, coins)         -- a list
Net.CoinsChanged.FireExcept(player, coins)         -- everyone but one
Net.CoinsChanged.FireNear(position, 100, coins)    -- characters within 100 studs
Net.CoinsChanged.FireGroup(group, coins)           -- a network group, or its name
```

Only for some options:
- `Event.FireFrom(position, ...)`: `Lod` events; each player at their ring's rate, nothing beyond the last ring.
- `Event.Forget(key)` / `Event.Forget()`: `Delta` events; drops the state kept for an object (call it when the object is gone).
- `Event.Exact(key, on)` / `Event.Exact(on)`: `Predict` events; turns guessing off (`true`) while precise movement matters, such as a fight.

Client:

```lua
local Net = Branch.Network.Client.ShopService

local disconnect = Net.CoinsChanged.On(function(coins)
	print("coins", coins)
end)
local pos = Net.Mob.At(id) -- Predict events: the guess for right now (nil before the first update)
```

`On` returns a disconnect function. Events that arrived before the first listener are queued and delivered to it.

### Client → server events (`From = "Client"`)

```lua
-- client
Branch.Network.Client.ShopService.Ready.Fire()

-- server
Branch.Network.Server.ShopService.Ready.On(function(player)
	-- every argument after player was already validated against the declaration
end)
```

Everything a client fires in one frame leaves as one remote call, in order.

### Functions (`Return`)

```lua
-- server: one handler; setting it again replaces it
Branch.Network.Server.ShopService.Buy.On(function(player, itemId)
	return true
end)

-- client: sends at once and yields for the answer
local ok = Branch.Network.Client.ShopService.Buy.Invoke("sword")
```

`Invoke` errors if the server's handler errored or the request could not be queued. Requests that arrive before a handler is set are answered once it is. A broken guard or rate limit answers "failed" (the `Invoke` errors).

### Network groups (server)

```lua
local group = Branch.Network.Group("round-1") -- the same name is always the same group
group:Add(player)       -- a player can be in many groups; adding twice changes nothing
group:Remove(player)
group:Has(player)       -- true / false
group:Players()         -- a copy of the list
print(group.Name)
```

Players leave every group when they leave the game; an empty group frees its memory.

### Refused packets (server)

```lua
Branch.Network.OnReject(function(player: Player, reason: string, forged: boolean)
	warn(`{player.Name}: {reason}{if forged then " (forged)" else ""}`)
end)
```

Called for packets that fail validation (sizes, ranges, NaN, wrong Instance classes, forged lengths, oversized packets; the packet is dropped), and at most once a second per player and event for rate limits (`rate limit:`) and broken guards (`cooldown:`, `not alive:`, `too far:`, `not theirs:`). `forged` is `true` for packets no Branch client can produce; after 3 of those (Settings › Network › Kick after forged packets) the player is kicked. Returns a function that removes the handler. Do not punish every rejection: values out of range or broken guards can come from lag or game bugs.

---

## 8. Network data types

Types go in `Data` and `Return`. They decide what travels and what the server accepts.

### Numbers

| Type | Bytes | Range |
|---|---:|---|
| `u8` / `u16` / `u32` | 1 / 2 / 4 | 0 to 255 / 65,535 / 4,294,967,295 |
| `i8` / `i16` / `i32` | 1 / 2 / 4 | ±127 / ±32,767 / ±2,147,483,647 |
| `f16` | 2 | ±65,504, about 3 significant digits |
| `f32` | 4 | about 7 significant digits |
| `f64` or `number` | 8 | a full Luau number |

```lua
"u8(0..100)"          -- range: the server refuses values outside it
"i16(-500..)"         -- -500 or more
"f32(..1)"            -- at most 1
"f32(0..1, 0.01)"     -- stepped: 101 steps, 1 byte; arrives rounded to the step
"f32(-180..180, 0.5)" -- 721 steps, 2 bytes
```

Out-of-range values error at the caller when argument checks are on (Settings › Network, Studio by default) and are clamped when off.

### Values

| Type | Carries |
|---|---|
| `boolean` | true / false (bit-packed with the event's other booleans) |
| `string`, `string(..32)`, `string(8)`, `string(4..16)` | text; max, exact or bounded length in bytes |
| `buffer`, `buffer(..512)` | a buffer, same length rules |
| `vector`, `vector(f16)` | a Vector3: three f32 (12 bytes) or three f16 (6) |
| `CFrame` | 24 bytes |
| `CFrame(compact)`, `CFrame(compact, f16)`, `CFrame(f16)` | 16 / 10 / 18 bytes; compact rotation about 0.1° off on average |
| `Color3` | 3 bytes |
| `Instance`, `Instance(Model)` | a reference (must exist on the receiving side); with a class, others are refused |
| `Enum(KeyCode)` | an EnumItem of that Enum; a mistyped Enum name is a compile error |
| `enum(Idle, Run, Jump)` | one of those strings, 1 byte |
| `unknown`, `unknown?` | any Roblox value, unchecked and uncompressed (`?` allows nil) |

### Containers and optionals

```lua
"u8[]"                          -- an array (also "{ u8 }")
"u8[..10]" / "u8[4]" / "u8[2..8]"
"{ [string]: u32 }"             -- a map (keys cannot be optional)
"{ id: u16, count: u8 }[..50]"  -- up to 50 structs
"vector?"                       -- optional
"(u8[])?"                       -- grouping
Data = { pos = "vector(f16)", state = "enum(Idle, Walk, Run)", target = "Instance(Model)?" }
Data = { "Instance(Player)", "f32(0..100, 0.5)" } -- a tuple
```

Clients may send arrays and maps up to Settings › Network › Max client array (16,384 by default).

Service, event and field names must be plain identifiers (`[A-Za-z_][A-Za-z0-9_]*`) and not Luau keywords.

---

## 9. Anticheat

```lua
local Anticheat = Branch.Network.Anticheat
```

On top of packet validation, every client signs a heartbeat and checks its own network module; strict mode (Settings › Network) can tag every packet. Each finding is a check name and an optional detail; what happens is up to `SetAction`. Never use the anticheat instead of server-side validation.

| Function | Side | Purpose |
|---|---|---|
| `Check(name, check, every?)` | client | Your own check: `check()` runs every `every` seconds (5 by default); returning a string (the detail) or `true` flags the player. Returns a remover. |
| `Report(check, detail?)` | client | Flags the player now from your client code. |
| `Flag(player, check, detail?)` | server | A finding from your server code. |
| `OnFlag(handler)` | server | `handler(player, check, detail?, score)` for every finding not set to `"ignore"`; returns a remover. |
| `Use(name, options? \| false)` | server | Ready-made movement checks: `"speed"` (`max`, `tolerance` 1.75), `"teleport"` (`distance` 100), `"fly"` (`height` 16, `time` 4), `"noclip"`; `false` turns one off. Off by default. |
| `Pause(player, seconds)` | server | Ready-made checks leave the player alone (your game moves them itself). Setting the character's CFrame from a script pauses them on its own. |
| `Preset("Basic" \| "Advanced")` | server | `"Basic"` (default): each check acts on its own. `"Advanced"`: checks only report, score 5 restricts, 10 kicks, 1 point a minute is forgotten. |
| `SetWeight(check, weight)` | server | Score per finding of a check (your checks: 1). |
| `SetLevel(at, action)` | server | When a score first reaches `at`: `"restrict"`, `"kick"`, a function `(player, record)`, or `nil` to remove. |
| `SetDecay(perMinute)` | server | Score forgotten per minute (Basic: 0). |
| `SetAction(check, action, after?)` | server | `"kick"` (default for every check but strict), `"report"` or `"ignore"`; with `after`, kicks from the after-th finding on. Kicks never happen in Studio. |
| `SetKickMessage(message)` | server | What kicked players see. |
| `Restrict(player, on?)` / `IsRestricted(player)` | server | A restricted player's refused requests count as forged and their rate limits refill at a quarter speed. |
| `GetFlagged()` / `GetRecord(player)` | server | Records: `{ player, score, restricted, checks = { [check] = { count, detail, at } }, history }` (frozen copies, last 50 findings). |
| `OnChanged(handler)` | server | `handler(player, record?)` on every record change (`nil`: gone); returns a remover. |
| `Pardon(player, check?)` | server | Forgives one check, or everything including a restriction. |
| `ShareAcrossServers(on?)` | server | Shares flagged players between the game's servers through MessagingService; call it on every server. |
| `GetFlaggedGlobal()` / `OnGlobalChanged(handler)` | server | Flagged players of every sharing server; `handler(userId, entry?)`. |

Configure checks conservatively: scripted movement, teleports, knockback, vehicles and abilities are legitimate; use `Pause`.

---

## 10. Included utility modules

Utilities are ModuleScripts in `Packages.Branch.Util`:

```lua
local Util = game:GetService("ReplicatedStorage").Packages.Branch.Util
local Scythe = require(Util.Scythe)
local ThreadReaper = require(Util.ThreadReaper)
local Timer = require(Util.Timer)
```

### Scythe: cleanup scopes

```lua
local scope = Scythe.scope()
Scythe.add(scope, part.Touched:Connect(onTouched)) -- :Disconnect()
local part = Scythe.add(scope, Instance.new("Part")) -- :Destroy(); returns the value
Scythe.add(scope, task.spawn(loop))                -- task.cancel()
Scythe.add(scope, function() end)                  -- called
Scythe.clean(scope)    -- dispose everything (newest first), keep the scope
Scythe.destroy(scope)  -- dispose everything and retire the handle
```

Also `Scythe.addBulk(scope, ...)` (all or nothing), `Scythe.remove(scope, value): boolean` (stop tracking without disposing), `Scythe.count(scope)`, `Scythe.isAlive(scope)`. Tables or userdata with `Destroy` or `Disconnect`, and other scopes, are cleaned too. Disposers must not yield.

### ThreadReaper: pooled threads

- `ThreadReaper.Spawn(callback, ...)`: now, in a pooled thread.
- `ThreadReaper.Defer(callback, ...)`: at the next resumption point.
- `ThreadReaper.Delay(seconds, callback, ...): token` (alias `Timeout`): later; a negative delay means now, NaN errors.
- `ThreadReaper.Cancel(token): boolean`: cancels a pending Delay.
- `ThreadReaper.Reserve(count)`: pre-creates pooled threads.

### Timer: countdowns

```lua
local round = Timer.new(90)      -- starts paused
round:OnFinished(function(state, seconds) end)
round:Start()                    -- alias Play: starts or restarts from the full length
round:Pause()
round:IncrementTime(15)          -- +15 s, running or paused
round:Resume()                   -- errors if not paused
local state, elapsed = round:Wait() -- alias Yield: "Finished" or "Stopped"
round:Stop()                     -- aliases Cancel, Kill
```

Reading: `GetLength()`, `GetRemaining()`, `GetState()`, `IsRunning()`, `IsPaused()`, `IsFinished()`, `IsStopped()`, `IsDead()`, `IsDied()`. Callbacks: `OnRunning`, `OnPaused`, `OnFinished`, `OnStopped`, `OnDead`, `OnDied` (each `fn(state, seconds)` in its own thread), and `timer.Callback`. States: `"Running"`, `"Paused"`, `"Finished"`, `"Stopped"`, `"Dead"`, `"Died"`; after a timer ends `GetState()` is `"Dead"`, while `Wait()` still says whether it finished or was stopped.

---

## 11. Types and autocomplete (2.5.2)

- **Both Luau type solvers.** `Branch.Network.Server` / `.Client` (every service, event, argument and return), `Branch.Data.Server` / `.Client`, `Branch.Keybinds` and `Branch:GetSegment(...)` are typed under the old and the new solver. Branch's own modules raise no type errors in Script Analysis. Write `--!strict` game code.
- **Segment types.** Branch Studio writes each segment's type into `Packages.Branch.Types` from the segment's code, and `Branch:GetSegment("ShopService")` returns it (`export type ShopService = ...` is also available from the Branch module). Methods' parameter types come from their annotations. Return types come from the annotation or, without one, are worked out from the `return` statements: literals, `self.Field` (fields assigned as `ShopService.Field = value`), other methods (`self:Other()`), local functions, `x :: T` casts, table literals (`{ id = id }` becomes `{ id: number }`), several values, `nil` on some paths (`T?`), no return (`()`).
- The segment module's own `type` / `export type` declarations (not generic ones) and `typeof(localFunction)` can be used in its annotations.
- What cannot be worked out stays `any`: values from other modules (`require`d helpers), method chains on other objects, generic methods. **Annotate the return type** of methods that other segments use and whose result is not plain from the code.
- Data paths and `Data.Get` results are typed from the data template when Branch Studio's Data types setting is on (point Settings › Project › Data template at the module that returns the template).

---

## 12. Branch Studio essentials

- **Compiles as you type:** after a pause in the script editor, Branch Studio rebuilds `Branch.Types`, the generated block inside the Branch module and `Branch.Network`. Nothing is written when nothing changed, and all of it is one undo step.
- **Problems** (Overview page): compile errors, network declaration errors with line numbers, unloaded modules that look like segments, template shape changes, keybind issues. Read them after changes.
- **Edits made from tools** (for example `ScriptEditorService:UpdateSourceAsync` or setting `.Source` through Studio automation) do not go through the script editor, so they do not trigger a compile on their own (adding or removing a script does). Compile explicitly: press the toolbar's Compile button, or from code with plugin permissions:

```lua
local widget = game:GetService("PluginGuiService"):FindFirstChild("BranchStudio")
widget:SetAttribute("BranchCommand", "compile")
-- afterwards: widget:GetAttribute("BranchCompileResult") is JSON:
-- { ms, segments, server, client, events, phases = { ... } }
```

  Other commands: `tab:<page>`, `install`, `perf`, `complete`, `section:<tab>`; see the Branch Studio guide's Automation section.
- **Services as you type:** typing a service the script has not defined (`require(Repli`) suggests it and adds its `game:GetService` line at the top of the script (Settings › Editor › Auto-import services). Code written by tools must still declare its own `local X = game:GetService("X")` lines.
- **Segment names as you type:** inside `GetSegment("…")` and `LocateSegments({ "…" })`.
- Branch Studio does nothing in Play mode; compile in Edit mode.

---

## 13. Security patterns

### Never trust the client

For every client action, the server checks:
- the player's data is loaded;
- the requested item, entity or action exists;
- the player may use it and is close enough (server-measured; `Near` / `Owned` guards help);
- cooldowns and rate limits (`Cooldown`, `Rate`);
- currency, prices, damage, rewards, inventory and progression are computed on the server;
- the player's current state allows it (alive, right round or area, not already claimed);
- repeated requests cannot double-spend or double-claim.

Branch validates the packet's shape, types, ranges and guards before any listener runs. That is not gameplay authorization.

### Secure purchase

```lua
--!strict
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local CATALOG: { [string]: { price: number, id: number } } = {
	sword = { price = 100, id = 1 },
}

local ShopService = {
	Dependencies = { "PlayerService" },
}

ShopService.Network = {
	Buy = { Data = "string(..32)", Return = "boolean", Rate = "4/s" },
}

function ShopService:Start()
	local Data = Branch.Data.Server
	local Net = Branch.Network.Server.ShopService

	Net.Buy.On(function(player, itemId)
		if not Data.IsLoaded(player) then
			return false
		end
		local item = CATALOG[itemId]
		if not item then
			return false
		end
		local coins = Data.Get(player, "coins")
		if type(coins) ~= "number" or coins < item.price then
			return false
		end
		-- checks first, then the writes (no yields in between)
		Data.Increment(player, "coins", -item.price)
		Data.Insert(player, "items", item.id)
		return true
	end)
end

return ShopService
```

The template's `items` must match what is inserted (here, numbers). Never accept a price, reward or result from the client.

---

## 14. Common mistakes and fixes

- **A segment never runs:** under `"Suffix"`, server names end with `Service` and client names with `Controller`; check the roots and `Branch:GetReport()`. Problems lists modules with `Init`/`Start` that are not segments.
- **`Branch.Network` errors:** the runtime is installed, the Network feature is on, the declarations compiled (Problems shows errors), and you use the right side.
- **Direction reversed:** `From = "Client"` is fired on the client and listened to on the server; `From = "Server"` the other way round.
- **A listener never fires:** check the declaration, the service name, that it compiled, and whether `Latest`, `Hz`, `Lod` or `Unreliable` explain delays or drops. Check `OnReject` for refused packets.
- **Data write errors:** wait for the data to load; check the path and that the value fits the template.
- **Data changed but did not save or replicate:** you changed a table from `Get`; use the write functions.
- **Template change warning:** add and test a migration.
- **Enum / class errors:** use exact `Enum(...)` names and real `Instance(ClassName)` classes.
- **Unknown network option:** the compile error suggests the right spelling.
- **`GetSegment` result is `any` for a method:** annotate that method's return type (§11).
- **Nothing compiles:** Branch Studio's Live toggle, the runtime installed and up to date, Studio not in Play mode; after tool edits, run the `compile` command (§12).
- **Versions:** check `Branch.Version`; press Update on the Overview when Branch Studio says the runtime is outdated.

---

## 15. Minimal example: data + event + controller

Branch Studio must compile the network declaration.

### `ServerScriptService/Services/PlayerService` (ModuleScript)

```lua
--!strict
local RunService = game:GetService("RunService")
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local PlayerService = {}

function PlayerService:Init()
	Branch.Data.Server.Start({
		Template = { coins = 0, lastClaim = 0 },
		Mock = RunService:IsStudio(),
		Leaderstats = { coins = "Coins" },
	})
end

return PlayerService
```

### `ServerScriptService/Services/RewardService` (ModuleScript)

```lua
--!strict
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local RewardService = {
	Dependencies = { "PlayerService" },
}

RewardService.Network = {
	Claim = { From = "Client", Rate = "2/s", Cooldown = 1 },
	CoinsChanged = { From = "Server", Data = "u32" },
}

local CLAIM_EVERY = 60 -- seconds

function RewardService:Start()
	local Data = Branch.Data.Server
	local Net = Branch.Network.Server.RewardService

	Net.Claim.On(function(player)
		if not Data.IsLoaded(player) then
			return
		end
		local now = os.time()
		if now - (Data.Get(player, "lastClaim") or 0) < CLAIM_EVERY then
			return
		end
		Data.Set(player, "lastClaim", now)
		local coins = Data.Increment(player, "coins", 10)
		Net.CoinsChanged.Fire(player, coins)
	end)
end

return RewardService
```

### `StarterPlayer/StarterPlayerScripts/Controllers/RewardController` (ModuleScript)

```lua
--!strict
local Branch = require(game:GetService("ReplicatedStorage").Packages.Branch)

local RewardController = {}

function RewardController:Start()
	local Net = Branch.Network.Client.RewardService

	Net.CoinsChanged.On(function(coins)
		print("coins:", coins)
	end)

	-- call from a real UI button:
	-- Net.Claim.Fire()
end

return RewardController
```

---

## 16. Official references

- [Branch home](https://praxprix-studios.github.io/Branch-Framework/website/index.html)
- [Getting started](https://praxprix-studios.github.io/Branch-Framework/website/getting-started.html)
- [Segments](https://praxprix-studios.github.io/Branch-Framework/website/guide/segments.html)
- [Player data guide](https://praxprix-studios.github.io/Branch-Framework/website/guide/data.html)
- [Networking guide](https://praxprix-studios.github.io/Branch-Framework/website/guide/network.html)
- [Branch Studio guide](https://praxprix-studios.github.io/Branch-Framework/website/guide/studio.html)
- [Troubleshooting](https://praxprix-studios.github.io/Branch-Framework/website/guide/troubleshooting.html)
- [Branch API](https://praxprix-studios.github.io/Branch-Framework/website/api/branch.html)
- [Server data API](https://praxprix-studios.github.io/Branch-Framework/website/api/data-server.html)
- [Client data API](https://praxprix-studios.github.io/Branch-Framework/website/api/data-client.html)
- [Networking API](https://praxprix-studios.github.io/Branch-Framework/website/api/network.html)
- [Network types](https://praxprix-studios.github.io/Branch-Framework/website/api/types.html)
- [Utilities](https://praxprix-studios.github.io/Branch-Framework/website/api/util.html)
- [Anticheat](https://praxprix-studios.github.io/Branch-Framework/website/anticheat.html)
- [Changelog](https://github.com/PraxPrix-Studios/Branch-Framework/blob/main/docs/markdowns/changelog.md)

If a signature or option is not in this file, check the official pages or the installed runtime instead of guessing.
