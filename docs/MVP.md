# Minimum viable product

## Purpose

The MVP is an open-ended development build for the local city simulation and read-only regional presence. It has no victory condition, Great Work or emergency economy.

The MVP evaluates:

- the operation and clarity of direct compact-town planning;
- road-network readability with exact commuter cars sharing queues with consequential freight;
- resource conservation and interaction in one oil production and freight chain; and
- read-only neighbour visits with no cross-city simulation.

## Player-facing scope

### Map, roads and construction

- One authored, mostly flat compact-town map with an orthogonal 64 by 64 prototype grid and provisional 16-metre cells.
- A coastline, visible oil deposits and a small number of fixed unbuildable obstacles.
- Camera pan, rotate, zoom and reset.
- One straight two-way street type, intersections created by the grid, a bulldozer and no terrain editing.
- A few fixed rectangular building footprints. Final map, cell and footprint dimensions are decided only after a documented scale test.
- Free movable blueprints, explicit payment to start, a very short visible construction phase and small demolition salvage.
- Direct placement only. There is no zoning or automatic building growth.

### Ordinary city buildings

- Homes, shops and ordinary factories.
- Gradual residential and workplace occupancy stored as building totals.
- Available jobs govern resident arrival, with a small bootstrap allowance.
- Shops provide fewer clean jobs; factories provide more jobs but reduce nearby residential tax.
- Paid player-triggered upgrades that require sustained healthy operation and keep the existing footprint. Buildings retain occupants and continue operating during the very short upgrade delay.
- One specific inactive or blocked reason in every inspector.

### Utilities and civic service

- Electricity is citywide with no distribution streets or range.
- One low-output unstaffed wind-turbine type and one larger staffed power-plant type. The plant needs roads for worker access; the turbine does not.
- Water travels through connected streets from a small unstaffed starter tower or a larger staffed extraction facility.
- Multiple starter towers and turbines are allowed. Their capacities support initial development; costs and footprints are tuning data.
- All other special facilities require workers. Partial staffing reduces efficiency; fire staffing affects response time.
- Circular factory ground pollution reduces local water extraction and fades gradually when the factory stops operating or is removed.
- Gradual resident loss during water shortage only. Unemployment, power loss and road loss do not evict existing residents.
- A visible fire-risk meter and one road-distance fire-station coverage system.
- Guaranteed protection from destruction inside valid coverage.
- Burning buildings stop operating. Uncovered fires have a visible response window before destruction; restoring coverage in time saves them.
- Fire stations respond to multiple fires independently after visible countdowns. Partial staffing slows response without reducing coverage distance or its safety guarantee.
- Single-building fires, no fire spread, no fire-engine vehicles and rebuildable ruins.
- No invisible emergency electricity, loan, bankruptcy or finished recovery system.

### Traffic

- One derived commuter car per working resident.
- Exact routes and moving vehicles driven by a deterministic road graph.
- A visible congestion score based on the simulated trips.
- Commute arrival does not directly change employment, tax, staffing or wellbeing.
- Cars and oil trucks share road queues, so commuter congestion can delay production and export income.
- Exact production trucks whose arrivals do transfer conserved inventory.
- Placeholder vehicle visuals; unique car, truck and service assets are not required.
- No avenues, lane editing, traffic lights, parking, public transport, fire engines or ordinary shop deliveries.

### Economy

- One integer-credit city budget.
- Construction cost, resident tax, filled-job tax and upkeep for infrastructure and special buildings.
- A developer-only admin command that adds a chosen number of credits.
- Player-facing recovery is undefined. The admin command is excluded from balance rules.
- Research is deferred: no centre, points, research purchases or research requirements in the MVP.

### Oil production

- Oil deposits with visible, finite but long-lived reserves.
- One drill with extraction rate and output storage.
- One refinery with crude input, declared conversion recipe and product output storage.
- One shoreline cargo port with declared intake and export capacity.
- Exact truck cargo, destination reservations and delivery.
- All oil facilities require workers; partial staffing reduces extraction, processing or export throughput.
- Trucks unload into port storage without payment. Income occurs only when the port exports that stock.
- Loaded cargo is discarded and recorded as a loss if its destination is demolished or destroyed. Road blockage alone preserves it. Facility stock losses are also recorded.
- Visible reasons for blocked extraction, dispatch, delivery, processing and export.

All quantities, capacities and recipe rates use integers. No input or product may be created, consumed or credited twice.

### Regions and visits

- Private two-city regions, one owner per city and one active publication lease per city.
- Create region, copy invite, join, claim city, reopen and leave flows.
- Local saves remain authoritative for editing. Both standalone and region-bound cities can open, simulate, edit and save offline.
- Publishing, visiting and membership changes require connectivity. A publication lease never gates local play.
- Immutable read-only city snapshots publish after durable saves, on clean exit and at a configurable fixed batch cadence.
- A visitor loads the latest compatible snapshot and uses the normal camera, overlays and inspectors without edit controls.
- City owner, online/offline state, snapshot revision and last-published time are visible.
- No cross-city power, water, workers, jobs, money, goods, services, pollution or objectives.

## Technical scope

The simulation is plain C# and does not store live Unity scene objects. City rules and traffic advance at separate fixed deterministic rates. Presentation interpolates saved simulation state and never decides gameplay.

The regional service includes the Worker, per-region Durable Object, SQLite records for reliable action retries, authentication, publication leases and immutable R2 snapshot path. The MVP uses these components for membership, ownership and snapshot publication. General city-effect delivery and cross-city transaction infrastructure are deferred until an exchange is defined.

A published visit snapshot is not a cloud recovery save and cannot become an editable second authority.

## Explicitly out of scope

- Zoning, demand-driven construction and automatic building upgrades.
- Curved or elevated roads, avenues, lane tools, parking and public transport.
- Direct commute-arrival effects on jobs, taxes, land value or resident wellbeing. Freight delays from shared queues remain in scope.
- Ordinary retail freight, garbage trucks, police vehicles, ambulances and fire engines.
- Water pipes, sewage, garbage, police, health, education and parks.
- Production chains other than oil.
- Goods as construction ingredients.
- Research centres, points, unlocks and multiple specialization choices.
- Player-facing bankruptcy, loans, emergency funding and guaranteed recovery.
- A win condition, scenario ending, score target, civic project or Great Work.
- Any cross-city simulation or trade, including general effect delivery, effect acknowledgements and transaction replay.
- Public matchmaking, searchable regions, moderation, chat and competition.
- Multiple-device editing, save merging, player-hosted servers, consoles and mobile.

## Acceptance gates

### Scale and construction gate

- Camera controls, blueprint movement, road connection and funded construction work through the normal interface. An invalid plan reports its blocking condition.
- At least two meaningfully different layouts can support homes, jobs, utilities and fire protection.
- Fixed footprints, coastline access and obstacles remain readable from the normal play camera.
- The 64 by 64 grid and 16-metre cells are either supported by screenshots and notes or revised before content depends on them.

### Local city gate

- Resident arrival, workplace filling, taxes, upkeep, upgrades, power priority, water loss and nuisance are deterministic under recorded commands.
- A building with a problem shows one primary operational reason and any separate upgrade or risk reason.
- Factory placement creates a visible tradeoff between higher job value, residential tax and water-source effectiveness. Pollution remains visible after shutdown or removal and decays over time.
- Starter utilities allow a new city to attract residents without workers or circular supply requirements. Staffed utilities and oil facilities visibly lose throughput when understaffed.
- Electricity reaches the whole map without a road path; water serves only its connected street network. Staffed power plants still need worker road access.
- Unemployment, power loss and road loss retain residents; water shortages cause gradual departures.
- Upgrades retain their plots and occupants, continue existing operation during their very short delay and apply the new capacity once at completion.
- A building under continuous valid fire coverage cannot be destroyed by fire, including with partial station staffing.
- An uncovered fire stops operation and allows a visible response window. Restoring valid coverage in time saves it; otherwise it leaves rebuildable ruins. Slower station response never overrides valid coverage.
- The admin credit command is available only in development builds and records its use in the session log.

### Traffic gate

- Every working resident has one stable derived home/job assignment and one scheduled commuter trip when eligible.
- Vehicle routes, queue choices and congestion score replay identically from the same save and command stream.
- Representative presentation may interpolate or cull, but it cannot alter route, queue or score results.
- Removing a road invalidates or reroutes affected trips without losing vehicles or creating duplicate arrivals.
- Commute success does not directly change employment or tax. Shared car/truck queues delay freight and therefore can affect oil income.

### Oil-chain gate

- Drill extraction never exceeds remaining reserve or storage.
- A refinery cannot process missing or in-transit crude.
- A truck cannot reserve the same cargo or destination capacity twice.
- Road removal, full storage and temporarily inactive destinations produce stable blocked states with specific explanations.
- A demolished or ruined destination causes each inbound load to be discarded exactly once, recorded as a loss, with reservations released and the truck available again.
- Unloading at the port earns nothing. Exporting from storage credits exactly the exported quantity once.
- A conservation test accounts for every unit as underground reserve, facility inventory, vehicle cargo, processed consumption, exported product or explicit destroyed/discarded loss; reservations never count as extra stock.

### Save and recovery gate

- Roads, blueprints, construction, buildings, occupancy, staffing, utilities, ground pollution and decay, fire and response state, traffic assignments, vehicles, inventories, recorded losses, deposits, balances and stable IDs survive save/reload.
- Immutable local generations recover from an interrupted save without silently resetting the city.
- Unsupported save versions show an explicit error.
- Replaying a canonical save and command stream produces the same simulation and traffic hashes.

### Regional visit gate

- Two accounts can create or join one private region and claim different cities.
- Concurrent claims have one winner and a second publisher cannot acquire the same publication lease.
- Opening a region-bound city offline works. A disconnect or expired publication lease never pauses local editing or saving. Reconnection resolves pending publication results without accepting a stale head.
- Snapshot upload is immutable, checksummed and committed as the visible head only after the blob exists.
- A lost publication response can be retried without creating conflicting heads.
- A visitor can load the latest compatible snapshot while the owner is offline and cannot mutate it.
- Incompatible or corrupt snapshots fail clearly; last-published time and revision are always visible.

### Performance and delivery gate

- Canonical local and traffic fixtures are checked in with exact coordinates, IDs, camera state, settings and hashes before their budgets block a milestone.
- macOS development builds run from the first scene gate and validate the completed MVP. Windows remains a target, but its runtime validation is explicitly post-MVP and does not block these gates. Record Windows runtime status as unverified until it is tested.
- Edit Mode tests cover deterministic city rules, traffic, fire, production conservation, saves and migrations.
- Backend tests cover identity, membership, claims, publication leases, idempotent publication, snapshot integrity and access control.

## Milestones

[The roadmap](ROADMAP.md) owns the milestone order and completion points. The local town, traffic and oil gates must pass before regional work. Read-only neighbour visits remain required to complete the MVP. Deeper transport and traffic are the first priority afterward.

## Pending decisions

- Final cell, map and footprint dimensions.
- Exact costs, taxes, upkeep, capacities, risk rates, construction times, staffing efficiency, pollution decay and fire response rates.
- Exact traffic step, schedules, shared intersection queue policy and congestion formula.
- Worker allocation and utility input dependencies; pollution overlap and decay formulas; fire response interruption and risk reset.
- Exact oil recipe, truck capacity, dispatch and truck reuse policy, deposit size and export price.
- Snapshot batch cadence and size budget.
- Player-facing failure, recovery and long-term progression.
- The first meaningful research branches and post-oil production chain.
