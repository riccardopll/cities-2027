# Cities 2027

*Cities 2027* is the working title for a stylized modern city-building game influenced by *SimCity* (2013).

The player controls the city as a whole. Player actions include planning roads, buying and placing buildings, providing utilities, managing money and developing specialized industries. The target audience includes new and experienced city-building players. The interface explains each system when it becomes relevant.

Standalone and region-bound cities support solo and offline local play. Multiplayer provides city visits in the MVP. Specialization and cooperation may be added later.

## Product priorities

Use this priority order when features conflict:

1. Clear city problem-solving through visible costs, capacities, routes and spatial consequences.
2. Direct player agency over construction and upgrades.
3. A readable compact town whose systems can be understood from the map and inspectors.
4. Regional features that do not require simultaneous player activity.
5. A technical foundation that can support later trade and cooperation without replacing the backend.

## Current MVP

The MVP is an open-ended development build. It has no victory condition, Great Work or custom objective.

The local build includes:

- a compact orthogonal road grid on a mostly flat authored map with a coastline, oil deposits and a few obstacles;
- directly placed homes, shops and ordinary factories with fixed footprints, very short construction and upgrade delays, with existing operation maintained during upgrades;
- population, local jobs, money, citywide electricity and street-distributed water;
- small unstaffed starter utilities, larger staffed utilities and partial-staffing efficiency;
- circular ground pollution with gradual decay, fire response windows and guaranteed safety inside valid staffed coverage;
- exact commuter cars and oil trucks sharing queues: commuter arrivals do not determine jobs or tax, but congestion can delay freight and export income;
- one staffed oil chain from deposit to drill, refinery, truck and cargo port storage, with payment on export and explicit accounting for destroyed or discarded cargo;
- local saves, developer diagnostics and an admin tool that can add credits during development;
- private two-city regions whose latest published city snapshots can be explored read-only.

The MVP has no cross-city electricity, workers, goods, money, services or simulation effects. Research is deferred until useful unlocks exist. Regional trade rules and general city-effect delivery remain future work.

## Delivery sequence

[The roadmap](ROADMAP.md) owns the detailed delivery order: settle required rules, validate scale and construction, build the local town, add shared traffic and the oil chain, pass local integration, then implement regional publication and visits. Saves, diagnostics and macOS checks grow with each milestone. Windows remains a target, with runtime validation deferred until after the MVP.

Neighbour visits remain part of MVP completion. Deeper transport and traffic are the first post-MVP priority. Research, additional specializations and regional trade follow only when their gameplay is defined.

## Documents

- [Roadmap](ROADMAP.md) defines delivery order, MVP completion and proposed future phases.
- [MVP](MVP.md) defines the exact test-build scope, acceptance criteria and gates.
- [Gameplay](GAMEPLAY.md) describes player actions, choices and consequences.
- [Simulation specification](SIMULATION_SPEC.md) defines deterministic local rules and required tests.
- [Technical specification](TECHNICAL_SPEC.md) defines code boundaries, persistence, publishing and verification.
- [Network specification](NETWORK.md) defines regions, identity, leases and visitable city snapshots.
- [Engine](ENGINE.md) records the Unity direction, presentation rules and asset policy.

## Explicitly deferred product decisions

- Player-facing failure, bankruptcy, loans and recovery.
- A finite scenario structure, victory condition or long-term city lifecycle.
- Cross-city trade, commuters, services, pollution, gifts and Great Works.
- Detailed production chains beyond oil and how later goods contribute to trade or projects.
- Traffic effects on employment, wellbeing, deliveries outside production, public transport and service vehicles.
- Public matchmaking, competitive play, city replacement and region size beyond the private two-city test.
- The shape of the research tree once more than one meaningful choice exists.
