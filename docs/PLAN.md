# Cities 2027

Working title for a stylized modern city-building game inspired by the readable city systems and regional presence of *SimCity* (2013).

The player represents the city as a whole rather than the literal powers of a mayor. They plan roads, buy and place buildings, provide utilities, manage money and develop specialized industries. The game is aimed at the same broad audience as *SimCity* (2013): interested newcomers should be taught clearly, while city-building fans should still find meaningful tradeoffs.

The city must be satisfying without another human player. Multiplayer adds presence, inspiration and, after the MVP, opportunities for specialization and cooperation. It is not an admission requirement for the local city game.

## Product priorities

When features conflict, protect these priorities in order:

1. Clear city problem-solving through visible costs, capacities, routes and spatial consequences.
2. Direct player agency over construction and upgrades.
3. A readable compact town whose systems can be understood from the map and inspectors.
4. Regional presence that enriches the city without making an absent player fatal.
5. A technical foundation that can support later trade and cooperation without replacing the backend.

Creative expression and multiplayer cooperation support the city-management game; they do not replace it.

## Current MVP

The MVP is an open-ended development build, not a shortened commercial game and not a scenario with an invented ending. The owner may evaluate it in sessions of roughly two to three hours, but the city has no victory condition, Great Work or custom objective.

The local build includes:

- a compact orthogonal road grid on a mostly flat authored map with a coastline, oil deposits and a few obstacles;
- directly placed homes, shops and ordinary factories with fixed footprints, short construction times and paid upgrades;
- population, local jobs, money, electricity, water and fire protection;
- exact commuter cars used to calculate a congestion score, without traffic affecting jobs or income yet;
- one conserved oil chain from deposit to drill, refinery, truck and cargo port export;
- a research centre that accumulates points as scaffolding for later population-and-research unlocks;
- local saves, developer diagnostics and an admin tool that can add credits during testing;
- private two-city regions whose latest published city snapshots can be explored read-only.

The MVP has no cross-city electricity, workers, goods, money, services or simulation effects. Regional trade rules remain a future design task.

## Delivery sequence

1. Prove camera controls, grid scale, fixed building footprints and placeholder presentation in a disposable Unity scene.
2. Build and test the plain-C# city simulation, versioned local save and developer inspection tools.
3. Make direct construction, occupancy, jobs, electricity, water, money and fire protection understandable locally.
4. Add deterministic road movement, exact commuter cars and a visible congestion score.
5. Add the oil production chain, inventories, real delivery trucks and cargo-port export.
6. Evaluate the local city in two-to-three-hour owner sessions before relying on backend work to create interest.
7. Deploy the future-ready regional service, publish immutable city snapshots and enable read-only neighbour visits.
8. Validate identity, recovery, performance and Windows/macOS builds as the corresponding systems become relevant.

Every step is a gate. Online visiting does not begin merely because the backend can be built, and later trading does not begin merely because the regional data model can support actions.

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
