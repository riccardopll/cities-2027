# MVP simulation specification

## Purpose

This document defines deterministic local rules for the MVP. The same initial save, ordered player commands and configuration must produce the same city and traffic state.

Numeric balance values remain editable tuning data until the relevant prototype gate passes. Structural rules, units and ordering belong in code and tests rather than Unity scene objects.

## World model

- The starting hypothesis is a 64 by 64 orthogonal grid with 16-metre cells.
- The scale spike may revise those dimensions before content and save fixtures are locked.
- The map is mostly flat and authored. It contains a regional road entrance, a coastline, oil-deposit cells and a small set of unbuildable obstacles.
- The grid does not wrap. Coordinates use a documented southwest origin and stable integer cell IDs.
- A street occupies grid cells and connects north, east, south and west.
- Buildings occupy fixed rectangular footprints defined in data. The MVP intentionally uses more than one footprint size.
- Each building footprint has declared road-access edges. The cargo port additionally requires a valid shoreline edge; an oil drill requires a valid deposit overlap.
- Every created world entity receives a monotonic stable city ID. Deleted IDs are never reused.
- Residents and jobs are integers stored on buildings and in assignments. They are not persistent citizen biography objects.

## Two fixed update rates

The simulation uses two deterministic clocks:

- A **city tick** advances population, utilities, fire risk, research, production and finance. The initial target is one in-game minute per city tick, one city tick per real second at normal speed and four per real second at fast speed.
- A **traffic step** advances vehicles, queues and arrivals in smaller increments. The initial implementation uses four fixed traffic steps per city tick.

Pause advances neither clock. Fast speed processes more complete fixed steps; it never enlarges the step or ties gameplay to rendered frame time. Presentation interpolates between traffic states and may cull vehicle models without changing the simulation.

## Between-step application pump

Work that must not occur halfway through a city tick or traffic step is queued and applied at a boundary:

1. Apply pause, speed and developer-control changes.
2. Apply validated player commands in their recorded order.
3. Apply completed save, identity or regional publication results that affect local status.
4. Rebuild dirty road, utility and coverage graphs.
5. Begin the next complete fixed simulation step.

Wall-clock completion order is recorded when asynchronous results enter the queue. Replays use that recorded order.

## Player commands

Saved world commands include:

- Place, move or remove an unfunded blueprint.
- Start construction for a valid blueprint.
- Demolish or rebuild a completed or ruined building.
- Place or bulldoze a street.
- Trigger a paid building upgrade.
- Enable or disable eligible infrastructure.
- Use a development-only command to add a specified number of credits.

A failed command changes nothing and returns one specific player-facing reason. The admin credit command is compiled or permission-gated out of release builds and writes a structured development log entry.

### Blueprint and construction states

A planned building moves through:

```text
blueprint -> underConstruction -> operating | inactive -> ruined
```

- Blueprints cost nothing and may move freely.
- Starting construction deducts the full integer construction cost once.
- Construction requires a valid footprint and the definition's declared prerequisites. Utility-source buildings must not require their own output and create circular placement rules.
- Construction advances by city ticks and completes once.
- Completed buildings cannot move.
- Demolition returns the definition's small salvage amount once.
- Rebuilding ruins uses a reduced declared cost and never restores old occupants or inventory from nowhere.

## Road access and layouts

Road connectivity is recalculated only when the street graph changes. A building has access when at least one declared access edge touches a street connected to the regional entrance.

Road layout affects construction cost, upkeep, frontage, routes and network-distance coverage. Traffic capacity is measured from simulated vehicles, but commute congestion does not yet feed back into the economy.

The MVP uses one two-way street definition. Curves, ramps, elevation, avenues, lane editing, signals and parking are not represented in saves or commands.

## Ordinary buildings, population and jobs

The minimum ordinary definitions are small homes, shops and factories plus their paid upgrades. Exact footprints and capacities are data chosen after the scale spike.

- A home has resident capacity and resident-tax value.
- A shop has clean job capacity and job-tax value.
- A factory has higher job capacity and job-tax value plus nuisance and water-contamination radii.
- Homes within a factory nuisance radius remain usable and upgradeable but use a lower declared resident-tax rate.

An ordinary building is operational only when construction is complete and its required road, electricity and water states are valid. Each building exposes one primary inactive reason in stable priority order. Upgrade eligibility is reported separately.

### Occupancy

Occupancy evaluates at a slower declared interval using one frozen snapshot:

1. Calculate active job capacity.
2. Calculate supported residents from active job capacity plus a small bootstrap allowance.
3. Add residents gradually to eligible homes in stable entity-ID order until capacity or supported population is reached.
4. Remove residents gradually from homes without water according to a declared drain rate.
5. Derive the active workforce as a declared fraction of active residents.
6. Fill active workplace jobs in stable proportional order.
7. Assign each filled job to one home worker using a deterministic matching rule.

Exact formulas and rates remain tuning data, but integer rounding and tie-breaking must be specified before a fixture hash is accepted.

### Upgrades

A player may pay for an upgrade only when:

- its population and research requirements are met;
- the building has remained road-connected, powered and watered for the required consecutive city ticks;
- occupancy is at or above the declared threshold; and
- the treasury can pay the full checked integer cost.

An upgrade keeps the stable entity ID, changes its definition and enters a short construction state. Requirements and failure reasons are previewed before payment.

## Electricity

Electricity is road-distributed citywide capacity.

1. Sum enabled, road-connected generator output.
2. Sum operating demand by category.
3. Allocate capacity in one documented fixed category order with stable entity ID as the final tie-breaker.
4. Mark unserved buildings inactive with `insufficientPower`.

The UI shows generation, demand, shortage and the exact priority order. Imported electricity, power offers, regional reservations and emergency-grid capacity do not exist in the MVP.

Generators have footprint, construction cost, output, upkeep and nuisance definitions. They do not require their own power to produce.

## Water

A road-connected water tower contributes declared gross capacity. Each nearby operating factory within its contamination radius applies a visible deterministic output penalty, capped so effective output never becomes negative.

1. Calculate gross tower output.
2. Apply contamination penalties.
3. Allocate effective capacity using a documented stable category order.
4. Stop new occupancy in an unserved building.
5. Drain existing occupants at the declared occupancy interval and rate.

The overlay shows tower capacity, contamination sources, effective output and unserved buildings. The MVP has no pipe graph, sewage or water-quality commodity.

## Fire

Each vulnerable completed building stores integer fire risk. Risk changes at a declared city-tick interval using visible definition values and current protection.

- A fire station must be operating and road-connected.
- Coverage follows shortest network distance, not straight-line radius.
- A covered building cannot transition to `ruined` because of fire.
- An uncovered building warns before its visible meter reaches the ignition threshold.
- At the threshold, that building alone becomes ruined. Fire does not spread.
- Ruin removes active occupants, jobs, tax and nonrecoverable contents. Rebuild rules are explicit.

No random source is required for MVP fire. If later tests add variation, it must use a saved deterministic stream and preserve the warning contract.

## Economy

All balances and rates are signed 64-bit integers with checked arithmetic. Network-facing values must remain within the agreed JSON integer encoding.

At each finance interval:

```text
income = activeResidentTax + filledOrdinaryJobTax + acceptedPortExports
costs = streetUpkeep + generatorUpkeep + waterUpkeep + fireUpkeep
      + researchUpkeep + productionFacilityUpkeep
availableCredits += income - costs
```

Construction, upgrades and rebuilding require sufficient credits. The simulation does not promise recovery from a negative or unproductive state. No automatic loan, invisible power, bankruptcy or reset rule exists.

The development admin command adds an explicit signed amount using checked arithmetic. It is not part of balance tests.

## Commuter traffic

Each filled job has one stable derived worker assignment from a home to a workplace. The assignment does not create a persistent citizen biography.

- Eligible workers receive deterministic departure times derived from saved configuration and stable IDs.
- One worker produces one commuter car trip from home access to workplace access and a corresponding return trip when scheduled.
- Vehicles choose a deterministic valid route. Equal routes use documented stable tie-breaking.
- Vehicle position, queue state and destination survive saves or are reconstructed only from sufficient deterministic saved state.
- Intersections and occupied road segments use one explicit queue/reservation policy; a vehicle may not occupy impossible space or pass through another vehicle.
- Trip time, waiting and failure contribute to a visible congestion score.
- Commute arrival does not change filled jobs, taxes, production or resident wellbeing in this MVP.

The simulation may render fewer vehicles for performance, but the hidden exact trips still determine the score. Decorative cars unrelated to the traffic state are forbidden.

## Oil production and freight

Every oil quantity is an integer and must exist in exactly one state.

### Deposit and drill

- A deposit stores finite `remainingCrude` and covers authored cells.
- A valid drill overlaps a permitted deposit area.
- At each recipe interval, an operating drill moves at most its extraction rate from deposit reserve to drill output storage.
- Extraction stops when the deposit or output capacity is exhausted.

### Refinery

- The refinery has separate crude-input and product-output storage.
- One declared recipe consumes exact crude units and produces exact product units after a declared processing duration.
- A cycle cannot start without all input or enough output capacity.
- Reserved and in-process units cannot be reserved again.

### Cargo port

- A valid cargo port occupies an approved shoreline footprint and has product intake storage and export throughput.
- Export removes accepted product once and adds the declared checked-integer credit value once.
- Product waiting in a truck or refinery earns nothing.

### Trucks and reservations

- A dispatch reserves exact origin cargo and exact destination capacity before creating a truck.
- The truck carries the reserved units over a deterministic road route.
- Arrival commits the transfer once. Cancellation or unreachable recovery releases both reservations once.
- A removed road may trigger deterministic rerouting. If no route exists, the truck enters a visible blocked state; cargo is not deleted.

The MVP must maintain this conservation identity:

```text
initialUndergroundCrude
= remainingUndergroundCrude
 + crudeInDrills
 + crudeReservedAtOrigins
 + crudeInTrucks
 + crudeInRefineryStorage
 + crudeInProcessing
 + crudeConsumedByCompletedRecipes
```

Completed product has a corresponding identity across refinery output, reservations, trucks, port storage and exported total. Conversion loss, if any, is an explicit recipe output difference rather than an untracked disappearance.

## Research

An operating research centre requires road access, electricity, water and filled jobs. It adds a declared integer number of research points at its interval and pays upkeep.

The MVP stores research points and exposes them in development UI but spends none. Oil buildings are available immediately. Future unlocks may require both a population milestone and a research cost; no tree or migration is defined until actual choices exist.

## Required deterministic tests

- Blueprint validation, construction payment, completion, salvage, ruins and rebuilding.
- Stable cell/entity IDs and fixed-footprint occupancy.
- Road connectivity, access edges and network-distance fire coverage.
- Population bootstrap, gradual occupancy, water drain, job filling and worker assignment.
- Factory nuisance tax and water-contamination output penalties.
- Electricity and water allocation ordering and inactive reasons.
- Fire warning, guaranteed covered protection, isolated destruction and no spread.
- Finance intervals, checked arithmetic and admin-command isolation.
- Two fixed update rates at pause, normal and fast speed.
- Commuter scheduling, route tie-breaking, queues, save/reload and congestion score replay.
- Oil extraction, recipe timing, storage, cargo reservation, rerouting, delivery, export and full conservation identities.
- Research production only while staffed and supplied.
- Save round-trip equality at every boundary used by tests.

## Performance fixtures

Performance budgets become release gates only after canonical fixtures are checked in with exact map data, entity IDs, routes, inventories, traffic schedules, camera transform, quality settings and SHA-256 hashes.

At minimum, maintain:

1. A local-city fixture large enough to exercise utilities, occupancy, fire coverage and overlays.
2. A commute fixture with enough exact cars to create queues.
3. An oil fixture with multiple active drills, refinery cycles, trucks and port exports.

Profile plain simulation, routing, queues, vehicle presentation, save size and snapshot size before adopting Burst, Jobs or ECS.
