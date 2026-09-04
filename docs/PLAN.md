# Cities 2027

*Cities 2027* is the working title for a stylized modern city-building game influenced by *SimCity* (2013).

The player controls the city as a whole. Player actions include planning roads, buying and placing buildings, providing utilities, managing money and developing specialized industries. The target audience includes new and experienced city-building players. The interface explains each system when it becomes relevant.

The local city supports solo play. Multiplayer provides city visits in the MVP. Specialization and cooperation may be added later.

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
- directly placed homes, shops and ordinary factories with fixed footprints, short construction times and paid upgrades;
- population, local jobs, money, electricity, water and fire protection;
- exact commuter cars used to calculate a congestion score, without traffic affecting jobs or income yet;
- one conserved oil chain from deposit to drill, refinery, truck and cargo port export;
- a research centre that accumulates points as scaffolding for later population-and-research unlocks;
- local saves, developer diagnostics and an admin tool that can add credits during development;
- private two-city regions whose latest published city snapshots can be explored read-only.

The MVP has no cross-city electricity, workers, goods, money, services or simulation effects. Regional trade rules remain a future design task.

## Delivery sequence

1. Prove camera controls, grid scale, fixed building footprints and placeholder presentation in a disposable Unity scene.
2. Build and test the plain-C# city simulation, versioned local save and developer inspection tools.
3. Make direct construction, occupancy, jobs, electricity, water, money and fire protection understandable locally.
4. Add deterministic road movement, exact commuter cars and a visible congestion score.
5. Add the oil production chain, inventories, real delivery trucks and cargo-port export.
6. Validate the integrated local city before starting regional implementation.
7. Deploy the future-ready regional service, publish immutable city snapshots and enable read-only neighbour visits.
8. Validate identity, recovery, performance and Windows/macOS builds as the corresponding systems become relevant.

Each step is a gate. Online visiting starts after the local gates pass. Trading starts after its gameplay rules are defined.

## Documents

- [MVP](MVP.md) defines the exact test-build scope, acceptance criteria and gates.
- [Gameplay](GAMEPLAY.md) describes the player experience and the broader roadmap.
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
