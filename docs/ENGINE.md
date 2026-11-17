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

### Unity project

- `game/` is the complete Unity project root, including `Assets/`, `Packages/` and `ProjectSettings/`.
- When scaffolding begins, create it from the minimal Universal 3D template matching the selected Unity 6.3 LTS patch. Retain the required URP configuration but remove template examples, tutorial content and unused packages.
- Set Unity asset serialization to Force Text and version-control metadata to Visible Meta Files. Commit every `.meta` file alongside its asset.
- Use `Zigrasite` as the internal codename and `com.riccardopll.zigrasite` as the application identifier. The public game name remains provisional.
- Use Microsoft's Unity extension for VS Code together with its C# and C# Dev Kit dependencies, and Unity's current `com.unity.ide.visualstudio` editor-integration package. Do not add the obsolete `com.unity.ide.vscode` package. Generated `.sln` and `.csproj` files remain ignored.

### Input and interface

- Use the Input System only. Do not add new code against the legacy Input Manager.
- The desktop control baseline is mouse or trackpad plus keyboard, expressed through input actions rather than device polling.
- Use UI Toolkit for runtime UI and editor tools. Add uGUI only if a concrete feature cannot be delivered suitably with UI Toolkit.

### Rendering

- Use URP and one controlled stylized material palette.
- Prefer readability of roads, access edges, building state, vehicle routes and overlays over visual density.
- Separate simulation snapshots from interpolated render transforms.
- A culled or simplified vehicle remains present in the traffic simulation.

### Cameras and modes

- The player starts in a small `Bootstrap.unity` scene. It constructs the application and infrastructure services once, remains loaded, and hosts a persistent UI Toolkit startup shell with Continue and New City actions; Visit is added when regional networking exists.
- The shell validates or creates the selected application state before it additively loads `City.unity`. A load failure remains in the shell with an explicit error instead of constructing a partial city view.
- `City.unity` contains the shared camera, world and UI presentation roots. It does not own or serialize the authoritative city state, and unloading it does not destroy that state.
- The editable city and a visited city use the same pan, rotate, zoom and reset camera controls.
- Both modes use the same city presentation. Mode-specific application state controls which tools and clocks are available; visit mode constructs no edit tools and advances no authoritative clocks.
- A later street-level visitor mode is not part of the MVP.

### World structure

- The starting scale is a 64 by 64 grid with provisional 16-metre cells.
- Roads occupy orthogonal cells.
- Buildings use a few fixed rectangular footprints with declared access edges.
- The first authored map is mostly flat and includes coastline, oil deposits and obstacles.
- Curves, bridges, tunnels, elevation tools and terraforming remain later work.

The scale test must record screenshots from normal and close cameras before dimensions become content constraints.

### Simulation presentation

- One thin Bootstrap `MonoBehaviour.Update` is the frame entry point: collect input and UI intent, enqueue application commands, admit completed asynchronous work at the between-step boundary, run whole city and traffic steps within the frame budget, publish an immutable presentation snapshot, then interpolate visuals.
- Do not use Unity `FixedUpdate` as either simulation clock. Its physics cadence and catch-up policy are not the game's authoritative time model.
- City rules advance on a fixed city tick.
- Vehicles and queues advance on a smaller fixed traffic step.
- Render frames interpolate between complete traffic states.
- Presentation events may animate construction, fire response and production but cannot change saved values.
- Electricity overlays show a citywide supply pool; water overlays show street connectivity. Ground-pollution overlays show circular affected areas and residual pollution after factory shutdown or removal.
- Local clocks and tools remain available offline. Publication-lease failures affect online status only.
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
- small wind turbines and water towers, larger staffed power and water facilities, and fire stations;
- oil drills, refineries and the cargo port;
- blueprints, construction sites, inactive and burning buildings, response countdowns and ruins.

Construction and upgrades have very short visible delays. MVP upgrades change appearance within the existing plot while preserving occupants and existing operation until completion. Research assets are deferred with research gameplay.

Fixed footprints and readable silhouettes have priority over visual variants. Asset catalogue work follows the scale, movement and production gates.

### Saving and published visits

- Save data, stable IDs and definition references, never live GameObjects.
- Write immutable local generations and migrate by writing a new generation.
- Build visit snapshots from a verified save boundary.
- Load visited cities into separate read-only state.
- A published R2 snapshot is not an editable cloud save.
- Standalone and region-bound local cities open without authentication; only online operations need it. A filesystem writer lock protects local editing. A regional lease protects publication, not the local simulation.

### Code structure

[TECHNICAL_SPEC.md](TECHNICAL_SPEC.md#unity-assembly-boundaries) owns the assembly dependency rules. Add each assembly with its first real code.

- `Simulation`, `Traffic` and `Application` use `noEngineReferences` and run in Edit Mode tests without a scene.
- `Infrastructure` implements external adapters; `Presentation` contains Unity views and tools.
- `Bootstrap` constructs concrete services and wires them manually. No dependency-injection framework is planned.
- The `Development` presentation assembly requires `ZIGRASITE_DEVTOOLS` and contains overlays and admin UI, not simulation rules.

File and network work use built-in C# tasks and cancellation under the [application scheduling rules](TECHNICAL_SPEC.md#fixed-step-scheduling). Coroutines are limited to presentation sequences.

### Runtime definitions

- ScriptableObjects may provide inspector-friendly authoring for numeric and catalogue data, but Bootstrap validates and converts them to immutable C# definitions before they enter `Application`, `Simulation` or `Traffic`.
- An immutable `MapDefinition` is the runtime boundary for authored map data.
- Defer whether maps are authored as ScriptableObjects, imported files or another editor workflow until the scale test exposes the data volume and editing needs.

## Development practices

- Develop and test daily on Apple Silicon macOS. Windows and macOS are the only current client targets; access to suitable Windows test hardware remains unresolved.
- Use Mono for routine development builds. Add a staged macOS IL2CPP smoke build at the local-integration gate.
- Once regional networking exists, Unity Editor and development builds use the latest remote development backend by default. A `ZIGRASITE_DEVTOOLS` selector may switch to the local Worker emulator; release builds do not construct this selector and will receive one fixed environment profile when a release environment exists.
- Commit only non-secret backend profile values such as endpoint URLs. Cloudflare deployment secrets and user credentials never belong in Unity assets or source control.
- Embed the Git commit SHA as the build identifier and expose it in development diagnostics. Append `-dirty` to the displayed development identity when tracked local files differ from that commit; persisted schema and rules versions remain independent integers.
- `ZIGRASITE_DEVTOOLS` is enabled only for Editor and Development configurations. Bootstrap does not construct development UI or admin command access without it. Admin commands are capability-checked and audited; development saves remain loadable when the capability is absent.
- When the first player build is required, add one checked-in C# build entry point to the `Zigrasite.Editor` assembly. Local menu commands and later Unity batch-mode automation call the same implementation for scene lists, target settings, development flags and build identity; output goes under ignored `game/Builds/`.
- Add continuous integration when the first automated tests exist. Build-runner and Unity licence setup remain deferred until then.
- Use structured logs for commands, fixed-step overruns, saves, route failures, cargo reservations and publications.
- macOS is the MVP validation platform. Windows remains a target, with runtime checks deferred until after the MVP; unresolved Windows access does not block the local-city or MVP gate. A macOS host may produce a Windows x64 Mono player, but this does not validate Windows runtime behavior. Windows IL2CPP builds also require a Windows build environment. Run integrated gameplay, save and visit checks when Windows testing begins.

## Initial asset sources

### Kenney

Suitable CC0 packs may provide placeholder roads, buildings, terrain, UI or nature props. Import only the files needed for the current gate.

### Quaternius

Quaternius CC0 city, vehicle and nature packs are deferred until placeholder traffic is functional.

### Asset rules

- Keep editable source art under `game/SourceAssets/`. Track both source-authoring binaries and exported non-text runtime media with Git LFS through explicit extension rules.
- Keep Unity YAML, `.meta` files, JSON, UXML, USS and source code in ordinary Git text; never place an entire directory under LFS indiscriminately.
- Put only exported, Unity-ready project assets under `game/Assets/`; do not make runtime content depend on source-authoring files.
- Use direct serialized asset references initially. Do not load project content through string-based `Resources` paths, and do not add Addressables until a defined loading or delivery requirement justifies it.
- Isolate third-party assets from project-owned assets, retain their licence and attribution records, and import only the files used by the game.
- Cell and footprint dimensions are independent of placeholder asset dimensions.
- Reuse a controlled colour palette and keep state overlays legible.
- Prioritize original roads, utilities, production buildings and specialization landmarks when custom art begins.

## Questions still to resolve

- Final Mono versus IL2CPP player-build policy after the staged macOS IL2CPP validation.
- Minimum Windows and macOS hardware targets.
- Intel Mac support and universal macOS builds.
- Final map, cell and building-footprint dimensions after the scale test.
- Exact camera limits and whether a later street-level visit mode is valuable.
- Vehicle presentation density and interpolation after traffic profiling.
- Final art palette, shoreline treatment and custom-asset workflow.
- Windows test-machine access.
- The frequency of macOS IL2CPP checks after the local-integration gate.
- Unity build runner and licence setup once continuous integration is introduced.
- Public-title or company-name changes after saves exist, including preservation or migration of Unity's persistent-data location.
- Whether mod support is ever required; there is no current public mod format or loader.

## References

- [Unity 6.3 LTS](https://unity.com/releases/unity-6)
- [Unity 6.3 manual](https://docs.unity3d.com/6000.3/Documentation/Manual/index.html)
- [Technical specification](TECHNICAL_SPEC.md)
- [Kenney assets](https://kenney.nl/assets)
- [Quaternius assets](https://quaternius.com/)
