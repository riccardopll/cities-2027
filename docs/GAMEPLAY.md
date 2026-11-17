# Gameplay

## Player role

The player controls a modern city as a whole. They plan streets, fund and place buildings, operate infrastructure and guide specialization through one city budget.

The main activity is solving city problems. The interface reports why a building is empty, a road is congested, water output has fallen or an oil delivery has stopped. The result of a player action is visible in the affected system.

## MVP loop

1. Lay out streets around fixed terrain, coastline and resource constraints.
2. Place movable, free blueprints for homes, shops, factories and infrastructure.
3. Connect a valid blueprint and explicitly pay to begin its short construction phase.
4. Start with unstaffed water towers and wind turbines, then supply roads, electricity and water so homes and workplaces can fill. Larger utilities need workers.
5. Balance homes with local jobs, then collect predictable resident and job taxes.
6. Add staffed fire protection; respond to uncovered fires before their visible destruction timer expires.
7. Watch commuter cars and trucks share road queues and improve layouts that delay deliveries or produce a poor congestion score.
8. Build and staff an oil drill, refinery and cargo port; move resources in real trucks and earn money when the port exports stored product.
9. Reinvest in construction and player-triggered upgrades.

The MVP is open-ended. It has no win screen, mandatory objective, Great Work or recovery guarantee. A development admin tool can add credits while balance and failure rules remain undefined.

## Construction and buildings

The MVP does not use zoning or automatic growable buildings. The player places every building directly.

### Blueprints and construction

- A blueprint is free and may be moved freely while unbuilt.
- Invalid or disconnected plans may remain on the map as blueprints.
- Starting construction is an explicit action. It requires sufficient credits and a valid footprint; ordinary occupied buildings also require the connections shown in their inspector.
- Construction takes a very short, visible amount of simulation time.
- Once construction meaningfully begins, the building cannot be moved.
- Demolition of a completed building returns a small salvage value. Exact percentages are tuning data.
- A destroyed building becomes ruins. Rebuilding the same site costs less than fresh construction.

### Ordinary building families

- **Homes** provide residential capacity and resident tax.
- **Shops** provide a smaller number of clean jobs and job tax.
- **Factories** provide more jobs and tax value, but nearby homes pay less tax because of nuisance.

Homes, shops and factories use a small set of fixed rectangular footprints. Paid upgrades increase capacity and change appearance within the existing plot; they do not claim extra land. Upgrade requirements include sustained road access, electricity, water and high occupancy. MVP upgrades have no research requirement. During the very short upgrade delay, occupants remain and the building keeps operating at its existing capacity; the new capacity takes effect at completion.

Occupancy changes gradually. Empty homes attract residents when the town has available local jobs, with a small bootstrap allowance so an empty map can start. Workplaces fill from the active resident workforce. The MVP stores residents and jobs as building totals; it does not simulate persistent household biographies.

Special facilities also need workers, except for the starter water tower and wind turbine. Partial staffing reduces efficiency: utilities supply less, oil facilities extract, process or export less, and fire stations respond more slowly. Zero workers means no operation for a staffed facility. Arrival of a commuter car does not determine staffing efficiency in the MVP.

Only water shortage makes existing residents gradually leave. Unemployment, lost electricity and lost road access can stop arrivals or activity but do not evict residents. Demolition and destruction still remove a building's occupants.

## Roads and traffic

MVP roads form an orthogonal grid and use one two-way street type. Curves, elevation, avenues, lane editing, parking, traffic lights and public transport come later.

Road layout matters through:

- construction and upkeep cost;
- buildable frontage and fixed building footprints;
- distance between factories and homes;
- routes to jobs and production destinations;
- authored obstacles, coastline access and the regional entrance.

One working resident creates one derived commuter car with a home, workplace and route. These cars are exact moving simulation objects during their trips, but the residents remain building-level assignments rather than persistent citizen agents. Jobs, taxes, staffing and wellbeing do not directly depend on commute arrival. Cars and oil trucks share road queues, so commuter congestion can still delay production deliveries and export income.

Production trucks are exact and consequential. A truck carries discrete cargo, follows a valid road route and transfers its load only when it reaches the destination. Placeholder boxes or capsules are sufficient until the movement and queueing rules work. Vehicle art must not determine the simulation.

## Electricity

- Electricity is citywide capacity with no distribution streets or range.
- An unstaffed wind turbine supplies a small amount and can operate anywhere its footprint is valid, without road access.
- A larger power plant needs workers and road access for those workers. Partial staffing reduces output. The road requirement belongs to the facility, not to electricity distribution.
- Both types have construction cost, capacity, footprint and upkeep. Multiple turbines are allowed; their low output, cost and space use govern their usefulness at scale.
- Imported power and regional power trading do not exist in the MVP.
- A shortage uses a fixed, visible category order rather than player-set priorities. Exact ordering for special facilities is tuning data and must be shown in the power UI.
- An unpowered building stays on the map and displays one specific inactive reason.
- There is no invisible emergency grid or special player-facing bailout. The development admin tool may add credits so another generator can be tested.

## Water

Water flows through connected streets without separate player-drawn pipes. An unstaffed starter water tower supplies enough for initial development. A larger water-extraction facility requires workers; partial staffing reduces output. Multiple starter towers are allowed. Water cannot pass between disconnected street networks.

Factories contaminate a circular area of the map. Water extraction inside that area is less efficient. Only operating factories emit pollution. Pollution is stored on the map and gradually fades when the factory shuts down, loses operation or is removed; these events do not instantly restore clean extraction. Exact pollution accumulation, overlap and decay rules remain to be defined.

When water is short, new occupancy stops and existing occupancy drains gradually with visible warnings. A shortage does not instantly destroy a building. Exact contamination distance, output and drain rate are tuning values.

## Fire protection

Fire is a transparent risk-management system, not a surprise disaster system.

- Each vulnerable building has a visible fire-risk meter.
- Risk rises predictably from its definition, operation and lack of protection.
- A supplied, staffed and road-connected fire station protects buildings within a network-distance limit.
- Valid coverage guarantees that a protected building cannot be destroyed by fire.
- A burning building stops operating. An uncovered fire has a visible response window before destruction, during which restoring valid coverage saves it.
- A station's response has a visible countdown. Fewer workers make it slower; they do not reduce coverage distance or remove the safety guarantee while coverage remains valid.
- A station responds to multiple covered fires independently, without a dispatch queue or engine fleet.
- A fire affects only its originating building; fire does not spread in the MVP.
- Destruction leaves rebuildable ruins and removes the building's active occupants and output.
- Fire engines are not simulated until the later service-traffic work.

The fire station has upkeep and provides protection. It produces no direct income.

## Economy

The game uses one integer-credit city budget.

- Ordinary occupied homes provide resident tax.
- Filled jobs in ordinary shops and factories provide job tax.
- Streets, utilities, fire protection and specialized production facilities create recurring costs.
- Specialized facilities may also create inventories, outputs and direct export income.
- Buildings and upgrades cost money only. Goods are not construction ingredients in the MVP.
- Population requirements may gate MVP upgrades. Research is deferred and does not gate MVP content.

Tax policy, bankruptcy, loans and recovery are undefined. The add-credits command is a development tool and is not a recovery mechanic.

## Oil specialization

Oil is the MVP's only production chain and is available from the start.

1. An oil drill occupies a valid oil-deposit site and extracts discrete crude-oil units into finite storage.
2. A refinery consumes exact crude units at a declared rate and produces exact refined-product units into separate storage.
3. A real truck moves crude from a drill to a refinery only when input capacity is reserved.
4. A real truck moves refined product from a refinery to the cargo port only when intake storage is reserved.
5. The cargo port occupies a valid shoreline site and stores delivered product. Its staffed export rate determines how quickly that stock is sold.
6. Credits are earned only when product leaves port storage through export. Truck unloading earns nothing.

Drills, refineries and ports all need workers. Partial staffing reduces throughput without changing recipe proportions. Units are accounted for: a facility cannot consume cargo still on a road, a truck cannot deliver twice, and destination storage cannot exceed capacity. If a destination is demolished or destroyed, its inbound loaded cargo is discarded, recorded as a loss and the truck becomes available again. A road blockage alone keeps cargo intact. Stock destroyed in a facility is also recorded as a loss.

Oil deposits are finite but should outlast several hours of ordinary development. Remaining reserves, discarded quantities and every blocked production reason are visible.

Future production chains may support local special buildings, regional trade and Great Works. Those rules are outside the MVP.

## Research

Research is deferred until useful unlocks exist. The MVP contains no research centre or points, and the oil chain is available immediately. Population and research may gate future advanced content once meaningful choices are defined.

## Regional presence

The commercial game remains solo-complete. In the MVP, a private region contains two independently owned cities. A player may open the latest published version of the neighbour's city using the normal pan, rotate, zoom, overlays and building inspectors, but cannot edit it.

Both standalone and region-bound cities can be played and saved offline. Publishing, visiting and regional membership changes require a connection. A network failure never pauses local simulation. The MVP does not support editing copied saves across devices or merging histories.

Published cities provide read-only regional context. The MVP does not rank them and contains no secrets, gifts, chat, shared objectives or cross-city simulation. Snapshots publish after durable saves, on clean exit and at configured batch intervals. The interface shows the last snapshot update time.

## Post-MVP direction

Deeper transport and traffic are the first priority after the MVP. Define the next road tools, traffic rules and consequences before selecting additional systems. Later candidates include ordinary freight, service vehicles, public transport, useful research, education, production branches, tourism and regional cooperation. Cross-city trade and the system that applies its transactions are deferred until an exchange is defined. Great Works require their own gameplay design. See [the roadmap](ROADMAP.md) for delivery order.

Major open product questions remain:

- What makes a city fail, recover or finish?
- How long should a city remain interesting?
- Which research branches and specializations create different strategies?
- How should traffic eventually affect jobs, wellbeing and land value?
- What, if anything, should cities exchange across a region?
- How many cities should a later region contain, and how are absent or abandoned owners handled?
