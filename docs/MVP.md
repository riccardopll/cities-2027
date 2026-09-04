# Minimum viable product

## Purpose

The MVP is an open-ended development build for the local city simulation and read-only regional presence. It has no victory condition, Great Work or emergency economy.

The MVP evaluates:

- the operation and clarity of direct compact-town planning;
- road-network readability with exact commuter cars and no economic congestion effects;
- resource conservation and interaction in one oil production and freight chain; and
- read-only neighbour visits with no cross-city simulation.

## Player-facing scope

### Map, roads and construction

- One authored, mostly flat compact-town map with an orthogonal 64 by 64 prototype grid and provisional 16-metre cells.
- A coastline, visible oil deposits and a small number of fixed unbuildable obstacles.
- Camera pan, rotate, zoom and reset.
- One straight two-way street type, intersections created by the grid, a bulldozer and no terrain editing.
- A few fixed rectangular building footprints. Final map, cell and footprint dimensions are decided only after a documented scale test.
- Free movable blueprints, explicit payment to start, a short visible construction phase and small demolition salvage.
- Direct placement only. There is no zoning or automatic building growth.

### Ordinary city buildings

- Homes, shops and ordinary factories.
- Gradual residential and workplace occupancy stored as building totals.
- Available jobs govern resident arrival, with a small bootstrap allowance.
- Shops provide fewer clean jobs; factories provide more jobs but reduce nearby residential tax.
- Paid player-triggered upgrades that require sustained healthy operation.
- One specific inactive or blocked reason in every inspector.

### Utilities and civic service

- Citywide road-distributed electricity capacity from placeable generators.
- Citywide road-distributed water capacity from one water-tower type.
- Factory contamination that visibly reduces nearby water-source output.
- Gradual occupancy loss during water shortage.
- A visible fire-risk meter and one road-distance fire-station coverage system.
- Guaranteed protection from destruction inside valid coverage.
- Single-building fires outside coverage, no fire spread and rebuildable ruins.
- No invisible emergency electricity, loan, bankruptcy or finished recovery system.

### Traffic

- One derived commuter car per working resident.
- Exact routes and moving vehicles driven by a deterministic road graph.
- A visible congestion score based on the simulated trips.
- Commute arrival does not yet change employment, tax, production or wellbeing.
- Exact production trucks whose arrivals do transfer conserved inventory.
- Placeholder vehicle visuals; unique car, truck and service assets are not required.
- No avenues, lane editing, traffic lights, parking, public transport, fire engines or ordinary shop deliveries.

### Economy and research

- One integer-credit city budget.
- Construction cost, resident tax, filled-job tax and upkeep for infrastructure and special buildings.
- A developer-only admin command that adds a chosen number of credits.
- Player-facing recovery is undefined. The admin command is excluded from balance rules.
- One staffed and supplied research centre that accumulates research points.
- No meaningful research purchase in this MVP and no special UI explaining the unfinished system. Points may appear in the ordinary inspector or developer diagnostics.

### Oil production

- Oil deposits with visible, finite but long-lived reserves.
- One drill with extraction rate and output storage.
- One refinery with crude input, declared conversion recipe and product output storage.
- One shoreline cargo port with declared intake and export capacity.
- Exact truck cargo, destination reservations and delivery.
- Export income only after the cargo port accepts product.
- Visible reasons for blocked extraction, dispatch, delivery, processing and export.

All quantities, capacities and recipe rates use integers. No input or product may be created, consumed or credited twice.

### Regions and visits

- Private two-city regions, one owner per city and one active editor lease per city.
- Create region, copy invite, join, claim city, reopen and leave flows.
- Local saves remain authoritative for editing.
- Immutable read-only city snapshots publish after durable saves, on clean exit and at a configurable fixed batch cadence.
- A visitor loads the latest compatible snapshot and uses the normal camera, overlays and inspectors without edit controls.
- City owner, online/offline state, snapshot revision and last-published time are visible.
- No cross-city power, water, workers, jobs, money, goods, services, pollution or objectives.

## Technical scope

The simulation is plain C# and does not store live Unity scene objects. City rules and traffic advance at separate fixed deterministic rates. Presentation interpolates saved simulation state and never decides gameplay.

The regional service includes the Worker, per-region Durable Object, SQLite action ledger and effect sequence, authentication, leases and immutable R2 snapshot path. The MVP uses these components for membership, ownership and snapshot publication. Later gameplay actions extend the same authority model.

A published visit snapshot is not a cloud recovery save and cannot become an editable second authority.

## Explicitly out of scope

- Zoning, demand-driven construction and automatic building upgrades.
- Curved or elevated roads, avenues, lane tools, parking and public transport.
- Traffic effects on jobs, taxes, land value or resident wellbeing.
- Ordinary retail freight, garbage trucks, police vehicles, ambulances and fire engines.
- Water pipes, sewage, garbage, police, health, education and parks.
- Production chains other than oil.
- Goods as construction ingredients.
- A usable research tree or multiple specialization choices.
- Player-facing bankruptcy, loans, emergency funding and guaranteed recovery.
- A win condition, scenario ending, score target, civic project or Great Work.
- Any cross-city simulation or trade.
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
- Factory placement creates a visible tradeoff between higher job value, residential tax and water-source effectiveness.
- A covered building cannot be destroyed by fire. An uncovered fire affects one building, leaves ruins and can be rebuilt.
- The admin credit command is available only in development builds and records its use in the session log.

### Traffic gate

- Every working resident has one stable derived home/job assignment and one scheduled commuter trip when eligible.
- Vehicle routes, queue choices and congestion score replay identically from the same save and command stream.
- Representative presentation may interpolate or cull, but it cannot alter route, queue or score results.
- Removing a road invalidates or reroutes affected trips without losing vehicles or creating duplicate arrivals.
- Commute success does not change employment or tax in this MVP.

### Oil-chain gate

- Drill extraction never exceeds remaining reserve or storage.
- A refinery cannot process missing or in-transit crude.
- A truck cannot reserve the same cargo or destination capacity twice.
- Road removal, full storage and unavailable destinations produce stable blocked states with specific explanations.
- The cargo port credits exactly the accepted export quantity once.
- A conservation test accounts for every unit as underground reserve, facility inventory, vehicle cargo, processed consumption or exported product.

### Save and recovery gate

- Roads, blueprints, construction, buildings, occupancy, utilities, fire state, research, traffic assignments, vehicles, inventories, deposits, balances and stable IDs survive save/reload.
- Immutable local generations recover from an interrupted save without silently resetting the city.
- Unsupported save versions show an explicit error.
- Replaying a canonical save and command stream produces the same simulation and traffic hashes.

### Regional visit gate

- Two accounts can create or join one private region and claim different cities.
- Concurrent claims have one winner and a second editor cannot acquire the same city lease.
- Snapshot upload is immutable, checksummed and committed as the visible head only after the blob exists.
- A lost publication response can be retried without creating conflicting heads.
- A visitor can load the latest compatible snapshot while the owner is offline and cannot mutate it.
- Older or incompatible snapshots fail clearly; last-published time and revision are always visible.

### Performance and delivery gate

- Canonical local and traffic fixtures are checked in with exact coordinates, IDs, camera state, settings and hashes before their budgets block a milestone.
- A macOS development build runs at the first scene gate. A Windows smoke build is due at the local-city gate, and both platforms are checked afterward.
- Edit Mode tests cover deterministic city rules, traffic, fire, production conservation, saves and migrations.
- Backend tests cover identity, membership, claims, leases, idempotent publication, snapshot integrity and access control.

## Milestones

1. **Scale spike.** Camera, grid roads, fixed footprints, coastline, obstacles and placeholder buildings in a disposable scene.
2. **Headless city kernel.** Commands, construction, occupancy, jobs, electricity, water, fire, economy, stable IDs and versioned saves.
3. **Local city presentation.** Tools, overlays, inspectors, warnings, research scaffolding and the admin credit command.
4. **Traffic prototype.** Separate fixed update rate, exact commuter cars, deterministic routing/queues and congestion score.
5. **Oil chain.** Deposits, facilities, inventories, exact trucks, cargo-port export and conservation tests.
6. **Local integration gate.** Validate the combined local systems and revise their rules before regional implementation.
7. **Regional foundation.** Authentication, two-city membership, claims, leases, durable future action infrastructure and immutable snapshot publication.
8. **Neighbour visiting.** Read-only loading, inspection, snapshot status and local failure-state validation.

## Pending decisions

- Final cell, map and footprint dimensions.
- Exact costs, taxes, upkeep, capacities, risk rates, construction times and research rates.
- Exact traffic step, schedules, intersection queue policy and congestion formula.
- Exact oil recipe, truck capacity, dispatch policy, deposit size and export price.
- Snapshot batch cadence and size budget.
- Player-facing failure, recovery and long-term progression.
- The first meaningful research branches and post-oil production chain.
