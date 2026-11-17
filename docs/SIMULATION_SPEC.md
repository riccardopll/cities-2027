# MVP simulation specification

## Purpose

This document defines deterministic local rules for the MVP. The same initial save, ordered player commands and configuration must produce the same city and traffic state.

Numeric balance values remain editable tuning data, evaluated at the relevant milestone. Structural rules, units and ordering belong in code and tests rather than Unity scene objects.

## World model

- The starting hypothesis is a 64 by 64 orthogonal grid with 16-metre cells.
- The scale test may revise those dimensions before content and save fixtures are locked.
- The map is mostly flat and authored. It contains a regional road entrance, a coastline, oil-deposit cells and a small set of unbuildable obstacles.
- The grid does not wrap. Coordinates use a documented southwest origin and stable integer cell IDs.
- A street occupies grid cells and connects north, east, south and west.
- Buildings occupy fixed rectangular footprints defined in data. The MVP uses more than one footprint size.
- Each building footprint has declared road-access edges. The cargo port additionally requires a valid shoreline edge; an oil drill requires a valid deposit overlap.
- Every created world entity receives a monotonic stable city ID. Deleted IDs are never reused.
- Residents and jobs are integers stored on buildings and in assignments. They are not persistent citizen biography objects.

## Two fixed update rates

The simulation uses two deterministic clocks:

- A **city tick** advances population, staffing, utilities, ground pollution, fire, production and finance. The initial target is one in-game minute per city tick, one city tick per real second at normal speed and four per real second at fast speed.
- A **traffic step** advances vehicles, queues and arrivals in smaller increments. The initial implementation uses four fixed traffic steps per city tick.

Pause advances neither clock. Fast speed processes more complete fixed steps; it never enlarges the step or ties gameplay to rendered frame time. Presentation interpolates between traffic states and may cull vehicle models without changing the simulation.

Local simulation and editing work offline, including for a city in a region. Authentication and publication failures change online status, not the city's clocks. No city system advances while the city is closed.

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
blueprint -> underConstruction -> operating | inactive
operating | inactive -> burning -> operating | inactive | ruined
```

- Blueprints cost nothing and may move freely.
- Starting construction deducts the full integer construction cost once.
- Construction requires a valid footprint and the definition's declared prerequisites. Utility-source buildings must not require their own output and create circular placement rules.
- Construction has a very short visible delay measured in city ticks and completes once. The exact duration is tuning data.
- Completed buildings cannot move.
- Demolition returns the definition's small salvage amount once.
- Rebuilding ruins uses a reduced declared cost and never restores old occupants or inventory from nowhere.

## Road access and layouts

Road connectivity is recalculated only when the street graph changes. A building has worker/resident access when at least one declared access edge touches a street connected to the regional entrance. The unstaffed wind turbine needs no road access. Water sources must connect to streets to distribute water. Electricity distribution does not use this graph.

Road layout affects construction cost, upkeep, frontage, water connectivity, routes and network-distance fire coverage. Cars and trucks share queues. Commuter congestion can delay production deliveries and export income, but commute arrival has no direct effect on employment, taxes or wellbeing.

The MVP uses one two-way street definition. Curves, ramps, elevation, avenues, lane editing, signals and parking are not represented in saves or commands.

## Ordinary buildings, population and jobs

The minimum ordinary definitions are small homes, shops and factories plus their paid upgrades. Exact footprints and capacities are data chosen after the scale test.

- A home has resident capacity and resident-tax value.
- A shop has clean job capacity and job-tax value.
- A factory has higher job capacity and job-tax value plus nuisance and a circular ground-pollution area.
- Homes within a factory nuisance radius remain usable and upgradeable but use a lower declared resident-tax rate.

An ordinary building is operational only when construction is complete and its required road, electricity and water states are valid. Each building exposes one primary inactive reason in stable priority order. Upgrade eligibility is reported separately.

### Occupancy

Occupancy evaluates at a slower declared interval using one frozen snapshot:

1. Calculate eligible job capacity across ordinary workplaces and staffed special facilities. Eligibility must not require jobs to be filled already.
2. Calculate supported residents from eligible jobs, the declared workforce fraction and a small bootstrap allowance.
3. Add residents gradually to eligible homes in stable entity-ID order until capacity or supported population is reached.
4. Remove residents gradually from homes without water according to a declared drain rate.
5. Derive the active workforce as a declared fraction of active residents.
6. Fill eligible workplace jobs in stable proportional order, including staffed utilities, fire stations and oil facilities.
7. Assign each filled job to one home worker using a deterministic matching rule.

Exact formulas and rates remain tuning data, but integer rounding and tie-breaking must be specified before a fixture hash is accepted.

Only water shortage causes gradual resident departure in the MVP. Unemployment, lost power and lost road access do not evict existing residents; they can block arrivals, active workforce participation or building operation. Demolition and destruction still remove occupants. Do not drain population merely because supported population falls below the current total.

### Special-facility staffing

- Every special facility requires workers except the starter water tower and wind turbine. Homes hold residents and are not workplaces.
- The MVP also includes a larger staffed water-extraction facility and a staffed power plant. Any number of starter towers and turbines may be built; their costs, footprints and low capacities are tuning data.
- Partial staffing reduces utility output and oil extraction, processing or export efficiency. Zero staffing gives no staffed-facility output. Recipes still consume and produce whole declared quantities; staffing changes throughput, not the recipe ratio.
- Fire staffing changes response time, not coverage distance or the guarantee against destruction inside valid coverage.
- Worker assignment, utility demand and production output are distinct states. Assignments are not conditioned on successful commute arrival.
- Before implementation, define the worker-allocation order, required utility inputs and integer efficiency formulas. The startup test must demonstrate that the unstaffed tower and turbine support homes and initial jobs without requiring workers who cannot yet arrive.

### Upgrades

A player may pay for an upgrade only when:

- its declared population requirement is met; MVP upgrades have no research requirement;
- the building has remained road-connected, powered and watered for the required consecutive city ticks;
- occupancy is at or above the declared threshold; and
- the treasury can pay the full checked integer cost.

An upgrade keeps the stable entity ID, footprint, access edges, occupants and existing operation during a very short visible delay. Store the paid pending upgrade and completion tick alongside the operating building; do not put it into the inactive new-construction state. At completion, apply the new definition and capacity once. Only one upgrade may be pending per building. It never claims neighbouring cells. Requirements and failure reasons are previewed before payment.

## Electricity

Electricity is one citywide capacity pool with no distribution streets or range. A disconnected consumer can receive power, although a separate road-access requirement may still prevent its operation.

1. Sum enabled wind-turbine output and eligible staffed power-plant output after staffing efficiency.
2. Calculate requested demand by category before allocating supply. Do not omit a request merely because the building was unpowered last tick.
3. Allocate capacity in one documented fixed category order with stable entity ID as the final tie-breaker.
4. Mark unserved buildings inactive with `insufficientPower`.

The UI shows generation, demand, shortage and the exact priority order. Imported electricity, power offers, regional reservations and emergency-grid capacity do not exist in the MVP.

Sources have footprint, construction cost, output and upkeep definitions. The wind turbine has low output, needs no workers and can operate on any valid footprint without a road. The larger power plant requires road access for workers. Neither requires electricity to generate electricity. Other utility inputs and nuisance values belong to their definitions and must preserve the startup path.

## Water

A starter water tower supplies a small amount without workers. A larger water-extraction facility requires workers and scales output with staffing. Both distribute water through connected streets; supply cannot jump between disconnected street components. There are no separate player-drawn pipes.

Factory pollution is persistent map state in a circular area around each factory. Water extraction in polluted cells is less efficient. Only operating factories emit pollution. Switching a factory off, losing operation, burning or removal stops its emissions; existing pollution fades gradually rather than disappearing. The exact accumulation, decay, overlap and extraction-footprint sampling rules remain to be defined. Store residual pollution independently of current factory operation so a shutdown cannot instantly restore source capacity.

1. Calculate each source's gross output after staffing where required.
2. Apply the source's local ground-pollution penalty, clamping effective output to a non-negative value.
3. Allocate effective capacity within each connected street component using a documented stable category order and requested demand calculated before allocation.
4. Stop new occupancy in an unserved building.
5. Drain existing occupants at the declared occupancy interval and rate.

The overlay shows street connectivity, source capacity, polluted areas, residual pollution after removal, effective output and unserved buildings. Pollution values and decay progress survive saves. The MVP has no separate pipe graph, sewage or water-quality commodity.

## Fire

Each vulnerable completed building stores integer fire risk. Risk changes at a declared city-tick interval using visible definition values and current protection.

- A fire station must be supplied, road-connected and have workers to provide valid coverage.
- Coverage follows shortest network distance, not straight-line radius.
- A covered building cannot transition to `ruined` because of fire.
- An uncovered building warns before its visible meter reaches the ignition threshold.
- At the threshold, that building enters `burning` and stops operating. Show a response window before destruction; fire does not spread.
- A station in range extinguishes a covered fire after a visible response countdown. Partial staffing increases response time but never causes a building under continuous valid coverage to be destroyed. Staffing does not shrink coverage distance. No fire-engine vehicle is simulated.
- Restoring valid coverage while an uncovered building is still burning saves it; extinguishing may take the station's response time, but the uncovered destruction deadline cannot override valid coverage.
- A burning building with no valid coverage becomes ruined when its response window expires.
- Ruin removes active occupants, jobs, tax and nonrecoverable contents. Rebuild rules are explicit.

A station handles multiple fires independently, each with its own response countdown; there is no station dispatch queue or engine fleet. Save risk, burning state and response countdowns. Before implementation, define risk reset after extinguishing and what happens if coverage is lost during response. These rules must preserve the covered-building safety guarantee.

No random source is required for MVP fire. If later tests add variation, it must use a saved deterministic stream and preserve the warning contract.

## Economy

All balances and rates are signed 64-bit integers with checked arithmetic. Network-facing values must remain within the agreed JSON integer encoding.

At each finance interval:

```text
income = activeResidentTax + filledOrdinaryJobTax + completedPortExportIncome
costs = streetUpkeep + generatorUpkeep + waterUpkeep + fireUpkeep
      + productionFacilityUpkeep
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
- Cars and production trucks share one explicit intersection and road-segment queue/reservation policy; a vehicle may not occupy impossible space or pass through another vehicle.
- Trip time, waiting and failure contribute to a visible congestion score.
- Commute arrival does not directly change filled jobs, taxes, staffing efficiency or resident wellbeing. Shared queues can delay freight, reduce supplied production and delay export income.

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
- Truck arrival transfers product into reserved port intake storage and earns nothing.
- Export removes stored product at the staffed port's export rate and adds the declared checked-integer credit value once. The next finance interval must not credit the same export again.
- Product waiting in a truck, refinery or port storage earns nothing.

### Trucks and reservations

- A dispatch reserves exact origin cargo and exact destination capacity before creating a trip.
- Loading moves units out of origin inventory into truck cargo and consumes the origin reservation. Loaded units are never simultaneously counted at the origin.
- The truck carries those units over a deterministic road route. Arrival transfers the load once and consumes the destination reservation.
- If the destination is demolished or ruined, discard the loaded cargo once, record an explicit loss with its reason, release the destination reservation and make the truck available again. A merely inactive or temporarily unreachable destination does not trigger disposal. Vehicle return/reuse movement remains to be defined; this rule does not authorize teleporting the truck.
- Cancelling an unloaded dispatch releases its reservations without destroying inventory. Destruction or demolition of facility stock records an explicit loss once, including reserved and in-process units.
- A removed road may trigger deterministic rerouting. If no route exists, the truck enters a visible blocked state; cargo is not deleted.

The MVP must maintain this conservation identity:

```text
initialUndergroundCrude
= remainingUndergroundCrude
 + crudeInDrillsIncludingUnloadedReservations
 + crudeInTrucks
 + crudeInRefineryStorage
 + crudeInProcessing
 + crudeConsumedByCompletedRecipes
 + crudeDestroyedOrDiscarded
```

Completed product has a corresponding identity across refinery output, trucks, port storage, exported total and destroyed or discarded product. Unloaded reservations are subsets of their facility inventory, not additional units. Conversion loss, if any, is an explicit recipe output difference rather than an untracked disappearance.

## Research

Research is deferred until useful unlocks are defined. The MVP has no research centre, points, upkeep, research gates or saved research state. Oil buildings are available immediately. Future research may use population milestones and costs; its rules are not part of this simulation contract.

## Required deterministic tests

- Blueprint validation, short construction/upgrade delays, one-time payment and completion, continued operation during upgrades, salvage, ruins and rebuilding.
- Stable cell/entity IDs and fixed-footprint occupancy.
- Road connectivity, access edges and network-distance fire coverage.
- Population bootstrap, gradual occupancy, water drain, job filling and worker assignment.
- Factory nuisance tax, circular ground pollution, decay after shutdown or removal and water-extraction penalties.
- Citywide electricity, street-connected water allocation, starter utility independence, partial staffing and inactive reasons.
- Resident departure from water shortage only; unemployment, power loss and road loss retain existing residents.
- Fire warning, burning response windows, independent simultaneous responses, staffing-dependent response time, guaranteed covered protection, isolated destruction and no spread.
- Finance intervals, checked arithmetic and admin-command isolation.
- Two fixed update rates at pause, normal and fast speed.
- Commuter scheduling, route tie-breaking, queues, save/reload and congestion score replay.
- Oil extraction, recipe timing, storage, cargo reservation, rerouting, delivery, export and full conservation identities.
- Shared car/truck queues, export payment after storage, and explicit cargo/stock loss without double accounting.
- Save round-trip equality at every boundary used by tests.

## Performance fixtures

Performance budgets become milestone gates only after canonical fixtures are checked in with exact map data, entity IDs, routes, inventories, traffic schedules, camera transform, quality settings and SHA-256 hashes.

At minimum, maintain:

1. A local-city fixture large enough to exercise utilities, occupancy, fire coverage and overlays.
2. A commute fixture with enough exact cars to create queues.
3. An oil fixture with multiple active drills, refinery cycles, trucks and port exports.
