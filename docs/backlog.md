# Backlog

Deliberately postponed. Do not implement any of this before the roadmap ([12-roadmap.md](12-roadmap.md)) is complete. Ideas only, not commitments.

## Monetization (first after the roadmap)

Monetize a finished game, not a prototype. Candidate list for when the time comes:
- **Game passes:** 2× Cash, VIP (exclusive skin, chat tag), Extra Fuel (PRO).
- **Developer products:** cash packs, Revive (continue a run once after being spotted).
- **Rules:**
  - `ProcessReceipt` lives in exactly one service;
  - a `purchases` data field holds processed receipt ids, so a purchase is granted once;
  - paid random items (crates) would require showing the odds.

## Tutorial (second)

- The first run is guided: a launch prompt, a ring path to the Ranger Cabin, "land on the table" and "freeze when the turret looks" prompts.
- The practice course becomes public (based on the flight gym).
- A fuel lesson when PRO unlocks.

## Content

- More zones with their own level gates (ideas: a coastal town like the reference, a desert base, a snowy radar station).
- More skins: lamp, bottle, cactus, traffic cone, baguette; PRO milestone skins (Golden Vase at 50 PRO strikes).
- Destructible Watchers in later zones.
- Wind on PRO summit sites.

## Systems

- Contextual disguise bonus: a skin hides better near matching objects (vase near tables and flower pots).
- Jet wash pushes light client-side props (`WashReactive` tag: cans, papers, leaves).
- Daily rewards, playtime rewards, group reward, a likes goal.
- Global leaderboards beyond PRO (total cash, stealth strikes).
- Friends: join PRO together (teleport a party), follow a friend into their server.
- PvP prop-hunt mode: some players fly rockets, others play guards.
- Analytics funnels (AnalyticsService): first launch, first kill, level 10, first PRO run.
- Open Cloud automation that publishes the place file to both places in one step.
- Chrono or another replication library, if profiling shows the custom replication is not enough.
