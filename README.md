# Fly to the Sky

A stud-style Roblox game. You are a rocket that hides in plain sight: fly through a rocky valley, slip into guarded outposts, freeze and pose as a flower vase when a guard looks your way, and strike the moment it turns away.

**Status:** design for the first playable zone is complete; implementation has not started. The game project itself lives in Roblox Studio on the developer's machine (Branch framework). This repository holds the design.

## Read in this order

| # | Document | What it covers |
|---|---|---|
| — | [CLAUDE.md](CLAUDE.md) | Rules for whoever (human or agent) implements this |
| 00 | [Overview](docs/00-overview.md) | Pitch, core loop, pillars, final decisions |
| 01 | [Reference](docs/01-reference.md) | Frame-by-frame breakdown of the reference video, what we take from it |
| 02 | [Architecture](docs/02-architecture.md) | Branch segments, folder layout, run lifecycle |
| 03 | [Network](docs/03-network.md) | Every `Service.Network` declaration |
| 04 | [Data & progression](docs/04-data-progression.md) | Data template, XP and levels, upgrades, rebirth |
| 05 | [Flight](docs/05-flight.md) | Controls, flight model, contacts, camera, tuning |
| 06 | [Effects](docs/06-effects.md) | Flame, jet wash, sound, stud style, performance |
| 07 | [Guards & targets](docs/07-guards-targets.md) | Guard AI, suspicion, targets, explosions, rewards |
| 08 | [PRO servers](docs/08-pro-servers.md) | Separate PRO place, teleports, release checklist |
| 09 | [Zone 1: Rocky Valley](docs/09-zone-1-rocks.md) | First playable zone |
| 10 | [Rocket model](docs/10-rocket-model.md) | Model specification |
| 11 | [Math](docs/11-math.md) | Every formula, with variables and purpose |
| 12 | [Roadmap](docs/12-roadmap.md) | Phases with acceptance criteria |
| 13 | [Reference physics](docs/13-reference-physics.md) | Frame-by-frame measurement of the reference flight; calibrates the flight defaults |
| — | [Backlog](docs/backlog.md) | Deliberately postponed: monetization, tutorial, and more |

## Key decisions at a glance

- **Framework:** Branch v2.5.2. Services and Controllers as segments, compiled networking ([reference](reference/Branch_API_Reference.md)).
- **Mode:** PvE. Guards are bots; players see each other's rockets.
- **Flight first:** the flight feel is the top priority. WASD rotates the rocket at any time, Space is thrust, and inertia does the rest. The rocket is kinematic, simulated on the client and validated by the server, and replicated through custom code on top of Branch networking (no Chrono for now).
- **PRO servers:** a separate place, open from level 10. Fuel exists only there, along with tougher guards and higher rewards.
- **First zone:** rocky nature (Rocky Valley), built from the first asset pack.
- **Later:** monetization and tutorial come after the core game is done ([backlog](docs/backlog.md)).

## Repository layout

```
README.md            this file
CLAUDE.md            rules for the implementer
docs/                design documents (read in numeric order)
reference/
  Branch_API_Reference.md   Branch framework API (v2.5.2)
  reference-video.mp4       the original gameplay reference
  frames.jpg                reference frames, 2 per second
  physics-measurements.csv  per-frame thrust and position measurements (docs/13)
```
