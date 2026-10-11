# 09 — Zone 1: Rocky Valley

The first playable zone, built from the first asset pack (rocky nature). Gameplay comes first: build it as a **greybox** from plain blocks, tune flight and guard sightlines, then dress it with the assets. This layout is a draft; adjust it to the actual assets.

## Theme

An alpine valley: grey cliffs and boulders, pine trees, grass meadows, a river with a waterfall, a quarry cut into the canyon floor. Bright day, light atmospheric haze for depth. Stud style on everything built from parts.

## Layout

```
                 [Summit Radar — PRO only, highest peak]
                               ▲
   [Hub + launch pad] ──200──► [A: Ranger Cabin] ──200──► [B: Cliff Outpost] ──250──► [C: Quarry Depot]
     cliff plateau              meadow                     cliff face                  canyon floor
            └──── river, waterfall, rock arch, narrow canyon shortcut ────┘
```
At an average of about 55 studs/s, sites are roughly 4, 8 and 12 seconds of flight from the pad.

## Hub (cliff plateau at the valley mouth)

- Spawn, launch pad (a single pad: rockets do not collide with each other), shop terminal, skin wardrobe, level leaderboard.
- On Main servers: the PRO portal with a "LVL 10" lock and the PRO leaderboard.
- A clear view down the valley, so the first launch shows the whole zone.

## Sites

### A — Ranger Cabin (easy; homage to the reference)

- A wooden cabin in a meadow; open windows on two sides, a front door, a chimney.
- Inside: one **Sentinel**, `IndoorTank` profile, Tank tier (100 HP), in the living room. Furniture: table, shelves, fireplace mantel, TV.
- **Perch spots:** windowsills, table, shelf, mantel, a flower pot by the door (the vase joke).
- **Approach:** open meadow with no guards outside. Free flight, learn the window entry.

### B — Cliff Outpost (medium)

- A bunker carved into the cliff face, with three entrances:
  1. the front gate, watched by a **Watcher** (`LedgeTurret`) on a rock ledge above it;
  2. a cave tunnel from the river side, dark, with a tight bend;
  3. a ventilation shaft from the clifftop: the sneaky route.
- Inside: **Sentinel** `HeavyTank`, Heavy tank tier (250 HP), plus one indoor Watcher covering the main hall.
- **Perch spots:** crates, barrels, lamp brackets, rock ledges, pipes.

### C — Quarry Depot (hard)

- An open pit on the canyon floor: rock piles, a conveyor belt, an excavator, a site office.
- A **Watcher** `Searchlight` tower: long range, narrow cone, wide sweep.
- Target: a **Bunker** (600 HP) with a small Sentinel turret on its roof.
- **Cover:** rock piles, the excavator arm, conveyor supports. Perching on the excavator in plain sight is the high-skill play.

### Summit Radar (PRO only, `ServerStorage.ProZones`)

- On the highest peak: "fly to the sky". A radar dish installation, Summit radar tier (1,500 HP).
- Two Watchers with overlapping but gapped cones; few perch spots (antenna masts, a fuel tank, a generator).
- On PRO servers fuel makes the climb itself a challenge: refuel stops are needed on the rock ledges along the way.

## Flight landmarks (fun between sites)

- A rock arch over the river to fly through.
- A waterfall with a gap behind it.
- A narrow canyon shortcut from the cabin toward the quarry, with an overhang.
- Tall pine clusters to weave through (CanQuery on trunks only, not on needles).

## Geometry rules

- **Solid = queryable.** Everything the rocket should hit has `CanQuery = true` and lives under `Zones/RockyValley/Geometry`. Grass, flowers, leaves and small decor have `CanQuery = false` and live under `Decor`.
- **Mesh rocks:**
  - `CollisionFidelity = PreciseConvexDecomposition` near flight paths and inside sites;
  - `Hull` for background rocks;
  - `Box` where a box is close enough.
  Spatial queries use collision geometry, so this decides what the rocket actually hits.
- **Openings:** windows and doors in sites A and B are at least 4 studs wide and 3 high (the fins span 2.1 studs). Interiors have ceilings of 10+ studs.
- **Perch surfaces:** flat (normal Y > 0.7) and at least 2×2 studs, so a landing is reliable.
- **Bounds:** an invisible zone volume around the valley rim, with a ceiling at 600 studs. Leaving it shows a warning and ends the run after 3 s (`OutOfBounds`).
- **Streaming:** `StreamingEnabled` off for zone 1 ([02](02-architecture.md#streaming)).

## Build order

1. Greybox: hub, pad, the three sites as blocks, rough cliffs from blocks or simple terrain.
2. Tune flight on the greybox (phase 2), then guard sightlines (phase 4).
3. Replace with assets site by site, re-checking collision fidelity and perch spots.
4. Lighting, atmosphere, sound ambience (wind, river, birds).
