# Gameplay

## Player role

The player controls a modern city as a whole. They plan streets, fund and place buildings, operate infrastructure and guide specialization through one city budget.

The main activity is solving city problems. The interface reports why a building is empty, a road is congested, water output has fallen or an oil delivery has stopped. The result of a player action is visible in the affected system.

## MVP loop

1. Lay out streets around fixed terrain, coastline and resource constraints.
2. Place movable, free blueprints for homes, shops, factories and infrastructure.
3. Connect a valid blueprint and explicitly pay to begin its short construction phase.
4. Supply road access, electricity and water so buildings can fill and operate.
5. Balance homes with local jobs, then collect predictable resident and job taxes.
6. Add fire protection before visible building risk becomes destructive.
7. Watch commuter cars use the road network and improve layouts that produce a poor congestion score.
8. Build an oil drill, refinery and cargo port; move conserved resources in real trucks and earn money only from delivered exports.
9. Reinvest in construction and player-triggered upgrades.

The MVP is open-ended. It has no win screen, mandatory objective, Great Work or recovery guarantee. A development admin tool can add credits while balance and failure rules remain undefined.

## Construction and buildings

The MVP does not use zoning or automatic growable buildings. The player places every building directly.

### Blueprints and construction

- A blueprint is free and may be moved freely while unbuilt.
- Invalid or disconnected plans may remain on the map as blueprints.
- Starting construction is an explicit action. It requires sufficient credits and a valid footprint; ordinary occupied buildings also require the connections shown in their inspector.
- Construction takes a short, visible amount of simulation time.
- Once construction meaningfully begins, the building cannot be moved.
- Demolition of a completed building returns a small salvage value. Exact percentages are tuning data.
- A destroyed building becomes ruins. Rebuilding the same site costs less than fresh construction.

### Ordinary building families

- **Homes** provide residential capacity and resident tax.
- **Shops** provide a smaller number of clean jobs and job tax.
- **Factories** provide more jobs and tax value, but nearby homes pay less tax because of nuisance.

Homes, shops and factories use a small set of fixed rectangular footprints. Larger versions are paid upgrades. Upgrade requirements include sustained road access, electricity, water and high occupancy. Population and research requirements will gate most future buildings and upgrades.

Occupancy changes gradually. Empty homes attract residents when the town has available local jobs, with a small bootstrap allowance so an empty map can start. Workplaces fill from the active resident workforce. The MVP stores residents and jobs as building totals; it does not simulate persistent household biographies.

## Roads and traffic

MVP roads form an orthogonal grid and use one two-way street type. Curves, elevation, avenues, lane editing, parking, traffic lights and public transport come later.

Even before traffic is coupled to the economy, layout matters through:

- construction and upkeep cost;
- buildable frontage and fixed building footprints;
- distance between factories and homes;
- routes to jobs and production destinations;
- authored obstacles, coastline access and the regional entrance.

One working resident creates one derived commuter car with a home, workplace and route. These cars are exact moving simulation objects during their trips, but the residents remain building-level assignments rather than persistent citizen agents. A failed or slow commute affects the congestion score only in the MVP; jobs, taxes and wellbeing do not yet depend on arrival.

Production trucks are exact and consequential. A truck carries discrete cargo, follows a valid road route and transfers its load only when it reaches the destination. Placeholder boxes or capsules are sufficient until the movement and queueing rules work. Vehicle art must not determine the simulation.

## Electricity

- Power is citywide capacity distributed through the connected road network, representing buried utility access.
- Generators are placeable special buildings with construction cost, capacity, footprint, nuisance and upkeep.
- Imported power and regional power trading do not exist in the MVP.
- A shortage uses a fixed, visible category order rather than player-set priorities. Exact ordering for special facilities is tuning data and must be shown in the power UI.
- An unpowered building stays on the map and displays one specific inactive reason.
- There is no invisible emergency grid or special player-facing bailout. The development admin tool may add credits so another generator can be tested.

## Water

The MVP has one water-tower type. A road-connected tower supplies citywide capacity without player-drawn pipes. Nearby factory contamination reduces its effective output, making source placement a different problem from buying electricity capacity.

When water is short, new occupancy stops and existing occupancy drains gradually with visible warnings. A shortage does not instantly destroy a building. Exact contamination distance, output and drain rate are tuning values.

## Fire protection

Fire is a transparent risk-management system, not a surprise disaster system.

- Each vulnerable building has a visible fire-risk meter.
- Risk rises predictably from its definition, operation and lack of protection.
- A road-connected fire station protects buildings within a network-distance limit.
- Valid coverage guarantees that a protected building cannot be destroyed by fire.
- An unprotected fire affects only its originating building; fire does not spread in the MVP.
- Destruction leaves rebuildable ruins and removes the building's active occupants and output.
- Fire engines are not simulated until the later service-traffic work.

The fire station has upkeep and provides protection. It produces no direct income.

## Economy

The game uses one integer-credit city budget.

- Ordinary occupied homes provide resident tax.
- Filled jobs in ordinary shops and factories provide job tax.
- Streets, utilities, fire protection, research and specialized production facilities create recurring costs.
- Specialized facilities may also create inventories, outputs and direct export income.
- Buildings and upgrades cost money only. Goods are not construction ingredients in the MVP.
- Population and research requirements may gate content, but they do not replace monetary construction costs.

Tax policy, bankruptcy, loans and recovery are undefined. The add-credits command is a development tool and is not a recovery mechanic.

## Oil specialization

Oil is the MVP's only production chain and is available from the start.

1. An oil drill occupies a valid oil-deposit site and extracts discrete crude-oil units into finite storage.
2. A refinery consumes exact crude units at a declared rate and produces exact refined-product units into separate storage.
3. A real truck moves crude from a drill to a refinery only when input capacity is reserved.
4. A real truck moves refined product from a refinery to the cargo port only when export capacity is reserved.
5. The cargo port occupies a valid shoreline site and exports at a fixed capacity.
6. Credits are earned only for product actually accepted by the port.

Units are conserved. A facility cannot consume cargo that is still on a road, a truck cannot deliver twice, and destination storage cannot exceed capacity. Oil deposits are finite but should outlast several hours of ordinary development. Remaining reserves and every blocked production reason are visible.

Future production chains may support local special buildings, regional trade and Great Works. Those rules are outside the MVP.

## Research

A supplied and staffed research centre consumes upkeep, electricity and water and accumulates research points over time. The longer-term design uses both population and research requirements to unlock most advanced buildings and upgrades.

The oil chain is available immediately. Research points may appear in the building inspector or developer diagnostics. The MVP has no research purchases or research unlock interface. A research tree requires at least two defined branches.

## Regional presence

The commercial game remains solo-complete. In the MVP, a private region contains two independently owned cities. A player may open the latest published version of the neighbour's city using the normal pan, rotate, zoom, overlays and building inspectors, but cannot edit it.

Published cities provide read-only regional context. The MVP does not rank them and contains no secrets, gifts, chat, shared objectives or cross-city simulation. Snapshots publish after durable saves, on clean exit and at configured batch intervals. The interface shows the last snapshot update time.

## Post-MVP direction

Later work may add avenues, commuter consequences, ordinary freight, service vehicles, public transport, additional utilities and services, education, production branches, tourism and regional cooperation. Cross-city trade, Great Works and multiplayer economics require separate designs and prototypes.

Major open product questions remain:

- What makes a city fail, recover or finish?
- How long should a city remain interesting?
- Which research branches and specializations create genuinely different strategies?
- How should traffic eventually affect jobs, wellbeing and land value?
- What, if anything, should cities exchange across a region?
- How many cities should a later region contain, and how are absent or abandoned owners handled?
