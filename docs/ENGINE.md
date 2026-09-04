# Engine

## Decision

Use Unity 6.3 LTS with the Universal Render Pipeline for the Windows and macOS client. Keep the authoritative city and traffic rules in plain C# assemblies that do not depend on Unity scene objects.

Unity is responsible for input, cameras, tools, 3D presentation, interpolation, overlays, inspectors, audio and builds. It is not the authority for whether a building operates, a car reaches work or a truck transfers cargo.

## Engine selection basis

- Mature desktop tooling and a short path to an interactive grid prototype.
- C# supports one language across Unity presentation and the testable simulation kernel.
- URP is suitable for the intended stylized, readable art direction.
- Editor tooling can visualize road graphs, footprints, coverage, queues and inventories while systems are still placeholders.
- Burst and the Job System remain available for measured simulation bottlenecks.

## Project direction

### Rendering

- Use URP and one controlled stylized material palette.
- Prefer readability of roads, access edges, building state, vehicle routes and overlays over visual density.
- Separate simulation snapshots from interpolated render transforms.
- A culled or simplified vehicle remains present in the traffic simulation.

### Cameras and modes

- The editable city and a visited city use the same pan, rotate, zoom and reset camera controls.
- Visit mode constructs no edit tools and advances no authoritative clocks.
- A later street-level visitor mode is not part of the MVP.

### World structure

- The starting scale is a 64 by 64 grid with provisional 16-metre cells.
- Roads occupy orthogonal cells.
- Buildings use a few fixed rectangular footprints with declared access edges.
- The first authored map is mostly flat and includes coastline, oil deposits and obstacles.
- Curves, bridges, tunnels, elevation tools and terraforming remain later work.

The scale spike must record screenshots from normal and close cameras before dimensions become content constraints.

### Simulation presentation

- City rules advance on a fixed city tick.
- Vehicles and queues advance on a smaller fixed traffic step.
- Render frames interpolate between complete traffic states.
- Presentation events may animate construction, fire, research and production but cannot change saved values.
- Use stable entity IDs to bind GameObjects to simulation data.

### Traffic visuals

The MVP needs accurate movement before a full vehicle catalogue.

- Use simple coloured boxes or capsules for commuter cars and oil trucks.
- Different colours and debug labels may distinguish commute, crude and product trips.
- Draw routes, destination reservations, queue position, speed and cargo in development overlays.
- Every displayed vehicle corresponds to an exact simulated trip.
- Fire engine, police car, garbage truck and public transport assets are deferred with their systems.

### Building visuals

Create or import only enough placeholder presentation to distinguish:

- homes, shops and ordinary factories and their upgrade states;
- generators, water towers, fire stations and research centres;
- oil drills, refineries and the cargo port;
- blueprints, construction sites, inactive buildings and ruins.

Fixed footprints and readable silhouettes have priority over visual variants. Asset catalogue work follows the scale, movement and production gates.

### Saving and published visits

- Save data, stable IDs and definition references, never live GameObjects.
- Write immutable local generations and migrate by writing a new generation.
- Build visit snapshots from a verified save boundary.
- Load visited cities into separate read-only state.
- A published R2 snapshot is not an editable cloud save.

### Code structure

- `Simulation` contains deterministic city rules.
- `Traffic` contains deterministic routes, queues and vehicle steps.
- `Application` coordinates commands, fixed steps, saves, leases, publication and visit mode.
- `Infrastructure` contains files, serialization, authentication, HTTP and clocks.
- `Presentation` contains Unity scenes, GameObjects and UI.

Assembly definitions enforce the dependency direction. `Simulation` and `Traffic` must run in Edit Mode tests without loading a scene.

## Development practices

- Keep numeric definitions in editable data assets converted to immutable runtime definitions.
- Use structured logs for commands, fixed-step overruns, saves, route failures, cargo reservations and publications.
- Build on macOS first, run the first Windows smoke test at the local-city gate and test both thereafter.

## Initial asset sources

### Kenney

Suitable CC0 packs may provide temporary roads, buildings, terrain, UI or nature props. Import only the files needed for the current gate.

### Quaternius

Quaternius CC0 city, vehicle and nature packs are deferred until placeholder traffic is functional.

### Asset rules

- Cell and footprint dimensions are independent of placeholder asset dimensions.
- Reuse a controlled colour palette and keep state overlays legible.
- Prioritize original roads, utilities, production buildings and specialization landmarks when custom art begins.

## Questions still to resolve

- Minimum Windows and macOS hardware targets.
- Intel Mac support and universal macOS builds.
- Final map, cell and building-footprint dimensions after the scale spike.
- Exact camera limits and whether a later street-level visit mode is valuable.
- Vehicle presentation density and interpolation after traffic profiling.
- Final art palette, shoreline treatment and custom-asset workflow.
- Unity build runner and licence setup for continuous integration.

## References

- [Unity 6.3 LTS](https://unity.com/releases/unity-6)
- [Unity 6.3 manual](https://docs.unity3d.com/6000.3/Documentation/Manual/index.html)
- [Technical specification](TECHNICAL_SPEC.md)
- [Kenney assets](https://kenney.nl/assets)
- [Quaternius assets](https://quaternius.com/)
