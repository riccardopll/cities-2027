# Technical specification

## Purpose

This document defines the code boundaries, persistence, protocol and verification strategy for the current game direction. Player-facing rules live in [GAMEPLAY.md](GAMEPLAY.md); exact local ordering lives in [SIMULATION_SPEC.md](SIMULATION_SPEC.md); deployed regional behaviour lives in [NETWORK.md](NETWORK.md).

## System overview

```text
Unity Bootstrap composition root
    -> wires presentation to application commands and queries
        -> plain-C# deterministic city simulation
        -> plain-C# deterministic traffic simulation
        -> save/publish coordinators
    -> selects Infrastructure adapters
        -> local immutable save generations
        -> IRegionGateway
            -> controlled responses in client tests
            -> HttpRegionGateway for deployed service

Single Cloudflare deployment
    -> Hono HTTP boundary
        -> regional application logic
            -> per-region SQLite Durable Object
            -> immutable opaque R2 visit snapshots
```

The local save owns the editable city, and local play works offline even after joining a region. The region owns identity, membership, claims, publication leases, publication heads and stored results for implemented operations. A publication lease controls online writes, never simulation or editing. A published city is a read-only projection, never a second editable authority. General city-effect delivery is deferred until a cross-city exchange is defined.

## Decisions

- Unity 6.3 LTS client for Windows and macOS.
- One repository with `game`, `backend` and `protocol` as the only implementation roots.
- C# simulation assemblies without dependencies on `UnityEngine`.
- Layered Unity assembly definitions with a small `Bootstrap` composition root.
- Orthogonal authored grid with fixed rectangular building footprints.
- Integer gameplay values and explicit stable ordering.
- Separate fixed city and traffic update rates.
- Local authoritative versioned JSON saves behind a persistence adapter, with immutable generations and migrations.
- One Cloudflare Worker deployment containing the HTTP entry point and SQLite-backed region Durable Object, with R2 for immutable visit snapshots.
- Hono is confined to the HTTP boundary; regional rules and persistence do not depend on it.
- Direct parameterized Durable Object SQLite statements and application-owned schema migrations; no ORM.
- Private two-city MVP regions with one active publication lease per city.
- JSON/HTTPS protocol with strict validation and limits until measurement justifies another encoding.
- Hand-maintained language-neutral protocol fixtures, with hand-written C# and TypeScript implementations and no generated protocol code.
- Private development uses one current unversioned HTTP contract, with no compatibility or negotiation machinery.
- Persistent action IDs, canonical hashes and stored results for reliable membership and publication retries. No generic city-effect stream or cross-city transaction framework in the MVP.
- No ECS, Burst, Jobs, binary protocol, WebSockets or peer-to-peer networking without a measured need.

## Repository layout

```text
/
|-- AGENTS.md
|-- docs/
|-- game/
|   |-- Assets/
|   |   |-- Zigrasite/
|   |   |   |-- Code/
|   |   |   |   |-- Simulation/
|   |   |   |   |   `-- Zigrasite.Simulation.asmdef
|   |   |   |   |-- Traffic/
|   |   |   |   |   `-- Zigrasite.Traffic.asmdef
|   |   |   |   |-- Application/
|   |   |   |   |   `-- Zigrasite.Application.asmdef
|   |   |   |   |-- Infrastructure/
|   |   |   |   |   `-- Zigrasite.Infrastructure.asmdef
|   |   |   |   |-- Presentation/
|   |   |   |   |   `-- Zigrasite.Presentation.asmdef
|   |   |   |   |-- Bootstrap/
|   |   |   |   |   `-- Zigrasite.Bootstrap.asmdef
|   |   |   |   `-- Development/
|   |   |   |       `-- Zigrasite.Development.asmdef
|   |   |   |-- Content/
|   |   |   |   `-- Definitions/
|   |   |   |-- Scenes/
|   |   |   |   |-- Bootstrap.unity
|   |   |   |   `-- City.unity
|   |   |   |-- UI/
|   |   |   |-- Art/
|   |   |   |-- Audio/
|   |   |   |-- Settings/
|   |   |   |-- Editor/
|   |   |   |   `-- Zigrasite.Editor.asmdef
|   |   |   `-- Tests/
|   |   |       |-- EditMode/
|   |   |       |   `-- Zigrasite.Tests.EditMode.asmdef
|   |   |       `-- PlayMode/
|   |   |           `-- Zigrasite.Tests.PlayMode.asmdef
|   |   `-- ThirdParty/
|   |-- SourceAssets/
|   |-- Packages/
|   |   |-- manifest.json
|   |   `-- packages-lock.json
|   `-- ProjectSettings/
|-- backend/
|   |-- src/
|   |   |-- http/
|   |   |-- region/
|   |   |   `-- migrations/
|   |   `-- index.ts
|   |-- test/
|   |-- package.json
|   |-- pnpm-lock.yaml
|   |-- tsconfig.json
|   `-- wrangler.jsonc
|-- protocol/
|   |-- contracts/
|   `-- fixtures/
`-- .gitignore
```

`Zigrasite` is the internal codename and therefore the stable prefix for first-party Unity folders, namespaces and assembly names. A later player-facing title does not require moving source files or changing code namespaces.

`game` is a complete Unity project: `Assets`, `Packages` and `ProjectSettings` are committed, including every Unity `.meta` file, while generated Unity directories such as `Library`, `Logs`, `Temp`, `UserSettings` and build output are ignored. Unity uses Force Text asset serialization and Visible Meta Files so scenes, prefabs, settings and ScriptableObjects remain reviewable. Editable source art and audio projects live under `game/SourceAssets`, outside `Assets`, so Unity imports only exported runtime assets. Exported first-party textures, meshes, materials and clips live under `game/Assets/Zigrasite`; imported third-party assets remain isolated under `game/Assets/ThirdParty`. Git LFS tracks explicit non-text source and runtime-media extensions in both asset roots; Unity YAML, `.meta`, JSON, UXML, USS and source files remain ordinary Git text.

The player build enters through `Bootstrap.unity`. Its Bootstrap component creates the application graph once and remains loaded. A persistent UI Toolkit shell offers Continue and New City, with Visit added when regional networking exists. It validates or creates the chosen application state before additively loading `City.unity`; failures stay in the shell rather than constructing partial Unity objects. The city scene owns cameras, world and in-city UI presentation objects but not authoritative simulation state. Both editable and visit modes use this scene with different application capabilities.

`backend` is one local pnpm package and one Cloudflare deployable unit. Its `package.json` exposes the stable local and CI commands `dev`, `check`, `test` and `deploy:dev`; Wrangler directly compiles, runs and deploys the API Worker. Do not add Vite or a custom bundler without a concrete requirement. The repository is a monorepo for coordinated source and contract changes, not a requirement for Unity and TypeScript to share a build system. Add owner-specific scripts beside the owning project when they are needed. Do not create a root JavaScript workspace or empty `tools`, CI or generated-code roots in anticipation of future work.

`protocol` contains short language-neutral documents for the current HTTP contract and manually authored exact JSON fixtures for valid, invalid, boundary and canonicalization cases. The C# and TypeScript DTOs and validators remain in `game` and `backend`; both implementations run the same fixture bytes. Editor-only Unity tests locate the repository root from the Unity project path, and backend tests use a repository-relative path, so the fixtures remain one source of truth outside `Assets` and are not shipped in player builds. JSON Schema or OpenAPI is not required, and the contract does not generate production code.

The tree is a placement rule, not a requirement to commit empty directories. Create each directory when its first real file exists.

Scaffolding follows delivery order. Create `game` first for the scale and local-city work. Do not initialize `protocol` or `backend` until the first real HTTP regional behavior is implemented; add those two roots together so the first contract document, shared fixtures, C# gateway behavior and TypeScript route agree. Until then, the roots remain absent rather than empty.

Every repository change is developed on a short-lived branch and merged into `main` through a pull request. A human reviewer is not required for this private single-owner repository; the pull request is the automated integration boundary. Add required checks only when the corresponding code and useful tests exist. `main` must remain deployable because a passing backend or protocol merge updates the remote development environment.

## Unity assembly boundaries

Each implemented runtime layer has its own assembly definition, using the `Zigrasite` prefix. The list below defines the intended dependency boundaries as the project grows. Create a layer when its first real code exists, and add references only when used; do not create empty assemblies or placeholder services to satisfy the full diagram during scaffolding.

- `Zigrasite.Simulation` has no project assembly or `UnityEngine` dependency.
- `Zigrasite.Traffic` references `Zigrasite.Simulation` and remains plain C#.
- `Zigrasite.Application` references Simulation, adds its Traffic reference when traffic coordination exists, remains plain C# and defines the use cases and ports used by outer adapters.
- `Zigrasite.Infrastructure` references Application and the domain assemblies whose persistence types it maps. It implements files, HTTP, authentication, clocks and content-authoring conversion.
- `Zigrasite.Presentation` references Application and contains Unity-facing interaction and rendering code.
- `Zigrasite.Bootstrap` is the composition root. It may reference every runtime layer to construct the object graph, but no other runtime assembly may reference Bootstrap.
- `Zigrasite.Development` contains development-only presentation, overlays and admin controls. A `ZIGRASITE_DEVTOOLS` define constraint excludes it from non-development builds.

Editor code and Edit Mode and Play Mode tests have separate assembly definitions. Test assemblies explicitly reference the layers they test; production assemblies never reference test or Editor assemblies. Assembly references point inward through the list above, with Bootstrap as the only place that selects concrete adapters.

Bootstrap constructs the Development UI and grants admin-command capability only when `ZIGRASITE_DEVTOOLS` is enabled in Editor or a Development build. Regional and local admin commands validate that capability and emit an audit record. Without the define, no development UI or command entry point is constructed. Save DTOs do not depend on the capability, so the same development save remains loadable without it.

## Client dependency boundaries

### Simulation

Owns deterministic city state and rules:

- grid, terrain, deposits, streets, blueprints, buildings and stable IDs;
- construction, upgrades within existing plots, occupancy, jobs, home/worker assignments, staffing, utilities, ground pollution, fire and money;
- oil inventories, recipes, reservations, export accounting and explicit resource-loss totals;
- serializable state and domain events.

It does not reference Unity objects, HTTP, authentication, file paths or rendered time.

### Traffic

Owns deterministic routes and moving trips:

- road graph projection and dirty rebuilds;
- commute schedules derived from the city simulation’s home/job assignments;
- route feasibility for production dispatch and movement of dispatched trucks;
- vehicle routes, fixed-step positions, queues, arrivals and congestion metrics;
- serializable or deterministically reconstructable traffic state.

Simulation owns worker assignments, facility inventories and cargo/storage reservations. Traffic reads those city facts and owns trip routes, queues and movement. Application coordinates dispatch at fixed boundaries; Simulation validates and commits inventory reservations. Traffic reports arrivals as immutable events, which Simulation validates before transferring cargo. Traffic never maintains a second copy of inventory ownership.

### Application

Coordinates use cases:

- validated player and admin commands;
- pause, normal and fast scheduling;
- fixed-step boundary application;
- save, load, migration and recovery flow;
- region create/join/claim/leave and publication-lease lifecycle, independent of offline editing;
- snapshot building, publication, download and visit-mode entry;
- durable outbox for actual membership/publication operations and their local completion status; no city-effect application.

### Infrastructure

Implements:

- atomic local files and immutable save generations;
- compression, hashes and canonical serialization;
- Unity Authentication adapter;
- `HttpRegionGateway` for the real backend; controlled gateway responses live in client tests, with no duplicate durable regional implementation;
- retry, timeout, backoff and clock adapters;
- structured logs and metrics.

### Presentation

Owns Unity scenes, GameObjects, cameras, tools, overlays, inspectors, audio and interpolated visuals. It issues commands and reads snapshots. It never mutates simulation collections directly or decides whether cargo arrived.

## Content definitions

Designers author building, recipe, vehicle and tuning definitions as Unity `ScriptableObject` assets under `Assets/Zigrasite/Content/Definitions`. During Bootstrap, an Infrastructure adapter validates and converts every asset into an immutable plain-C# simulation definition before the application permits a new game or load. Simulation and save state never retain a `ScriptableObject`, Unity object reference or asset GUID.

Definition IDs are manually assigned, stable and readable namespaced strings, for example `building.home.small` or `recipe.oil.refining`. Conversion rejects duplicate IDs, missing references and invalid values. Saved entities store these IDs plus any state that cannot safely be re-derived; lookup resolves the ID against the converted definition catalog.

Required definition categories include:

- homes, shops, ordinary factories and upgrades;
- streets, unstaffed wind turbines and starter water towers, staffed power plants and water-extraction facilities, and fire stations;
- oil deposits, drills, refineries, trucks and cargo ports;
- footprints, access edges, costs, capacities, upkeep, tax, staffing efficiency, nuisance, persistent pollution, coverage, risk and fire-response timings;
- recipes, storage, cargo sizes, processing durations and export values.

Content IDs are never derived from display names, Unity asset GUIDs or asset paths. A display name or file can change without changing identity. Renaming or removing an ID requires a save migration or an explicit incompatible test-save reset.

Runtime content uses direct serialized Unity references initially. Do not introduce string-based `Resources` loading or Addressables until a defined loading, memory or delivery requirement needs them. No mod loader, public content package or mod compatibility promise exists.

## Fixed-step scheduling

The application owns an accumulator for fixed city ticks and fixed traffic steps. Render frames consume immutable presentation snapshots and interpolate vehicle transforms.

- A thin Bootstrap `MonoBehaviour.Update` collects input and UI intent, queues application commands, admits completed asynchronous work only at a between-step boundary, runs complete fixed steps within budget, publishes one immutable presentation snapshot and then lets Presentation interpolate visuals.
- Infrastructure file, authentication and HTTP APIs use built-in `Task`, `async`/`await` and `CancellationToken`. Completion produces immutable result data queued for that boundary; continuations do not mutate simulation collections or Unity objects. Unity coroutines are reserved for presentation timing, and no third-party async package is introduced.
- Unity `FixedUpdate` does not drive city or traffic time. The simulation clocks remain explicit and independent of Unity's physics cadence.
- Pause processes UI, saving and networking but advances no simulation clocks.
- Normal and fast speed change how many complete fixed steps are processed.
- A maximum work budget prevents a slow frame from causing an unbounded catch-up spiral; hitting it produces a metric and visible development warning.
- A save is captured only at a complete declared boundary.
- Asynchronous results enter through the between-step pump and are recorded for replay.

Traffic results that transfer production cargo are emitted as immutable arrival events and applied exactly once at the next permitted city boundary.

## Save system

### Format and adapter

The current local-save representation is canonical, uncompressed UTF-8 JSON with an explicit `saveSchemaVersion`. Application code uses a local-save port; the Infrastructure file adapter owns paths, JSON DTO mapping, canonical encoding, checksums, atomic generations and recovery. Simulation types do not serialize themselves and do not depend on the filesystem representation. Add compression only after measured save size or I/O cost justifies it. This boundary permits a measured future encoding change without replacing simulation or application code.

Persistence DTOs are separate from runtime objects. Loading validates the envelope and complete DTO before constructing runtime state. A migration converts a validated older DTO to the next schema version; it never partially initializes live simulation state.

Select the concrete JSON implementation when the first save DTOs exist. It must support the required collections and canonical writer behavior; do not assume Unity's `JsonUtility` is suitable for complex DTO shapes such as dictionaries.

Unity Editor and development-player runs currently share one development save root under Unity's persistent-data location. Do not add copy or import tooling merely to separate them. An editable city must hold an exclusive local writer lock and fail clearly if a second process already owns it. Tests receive injected temporary roots and never access development saves. A release save root and its migration policy are defined only when a release identity exists. Because Unity derives desktop persistent-data paths from player settings, changing the company or player-facing product name after saves exist requires an explicit path-preservation or migration decision.

The save repository is keyed by stable city ID and may retain multiple editable cities, while Application loads at most one for editing. Each city owns its generations, head and writer lock. A small rebuildable catalogue records display metadata and the last-opened city; New City creates another entry and never overwrites an existing city. Continue resolves the last-opened entry to a verified generation before the city scene loads.

New City assigns a canonical random UUIDv4 before writing the first generation. This client-created ID survives later region claims and publications and is independent of the mutable city name. The backend atomically binds an unclaimed ID to its region and owner; a conflicting existing claim is an error and never triggers silent local-ID remapping.

Region, city and durable mutation-action IDs use the same lowercase hyphenated UUIDv4 representation, but every region, city and action receives its own value. They are the only client-created globally scoped identifiers and are not displayed in normal player UI. Do not add a Base58 codec for them. Deterministic entities inside a city use city-scoped counters instead. The backend assigns each HTTP attempt a diagnostic trace ID and returns it with errors; the client does not add a separate request ID to the contract.

### Envelope

Every save contains:

```text
format marker and saveSchemaVersion
gameBuildGitSha and contentManifestVersion
simulationRulesVersion and trafficRulesVersion
cityId, region reference and city seed
localSaveRevision
cityTick, trafficStep and speed state
next stable entity, vehicle and command IDs
map, roads, blueprints, buildings, pending upgrades and ruins
occupancy, job assignments, staffing and utility state
ground pollution and decay state
fire risk, burning, coverage and response countdowns
credit balances
oil deposits, inventories, recipes, reservations and explicit loss totals
vehicle routes, queues and cargo or sufficient reconstruction state
regional publication revision
pending regional action outbox
payload length and SHA-256 checksum
```

Runtime references, GameObjects, access tokens, lease tokens, invite secrets and storage credentials are forbidden.

### Immutable generations

Write a save to a unique temporary generation, flush it, verify length and checksum, then atomically advance a tiny head record. Keep several known-good generations and scan them newest-first when the head is missing or corrupt.

Never overwrite the only valid city file. A migration reads an old generation and writes a new generation; it does not alter the source.

Autosave cadence is tuning and must not stall fixed-step processing. Important construction, cargo and regional boundaries may request an immediate save through the coordinator.

### Pending membership and publication operations

Before its first send, an implemented regional mutation receives a UUIDv4 action ID and canonical payload and is saved in a persistent outbox. Retry the same bytes until the service returns its stored result. Persist the resulting local operation status before removing the outbox entry. Operations that precede a city claim use application-owned local persistence rather than requiring an already-claimed city save.

No generic client action sequence, ordered city-effect stream, effect acknowledgement or cross-city transaction replay is implemented or saved in the MVP. Incoming regional results update membership and publication status, not city gameplay. New offline saves coalesce into the latest eligible snapshot; already-sent actions retain their identity until resolved.

Local startup requires the city writer lock and a valid local save, not authentication or a publication lease. On reconnect, resolve pending results, validate ownership and the current publication head, and acquire or renew the publication lease before publishing. Rejection of a stale or conflicting head leaves local play running. Copied-save editing and history merging remain unsupported.

## Visit snapshot projection

The snapshot builder reads one verified local save boundary and emits only data required for read-only reconstruction. It removes secrets, admin history, unpublished actions and recovery internals.

Canonical serialization must provide:

- stable field and collection ordering;
- explicit schema, simulation, traffic and content versions;
- compressed and uncompressed length limits;
- SHA-256 for canonical and compressed bytes;
- deterministic output for identical source state.

The game owns the complete visit-snapshot schema, construction, validation and reconstruction. The uploaded compressed snapshot is an opaque byte object to the backend. The backend may enforce authorization, compressed-byte size, declared metadata and byte checksum, but it does not decompress, deserialize, migrate or extract gameplay fields from the snapshot. Snapshot format, simulation, traffic and content compatibility is checked by the downloading game before scene construction.

Visit mode loads the validated snapshot into separate read-only application state. Edit tools, simulation scheduling, local saving, admin commands and regional mutation commands are not constructed for a visited city.

## Regional protocol

### Route groups

The exact HTTP paths may evolve, but the current protocol requires route groups for:

- authentication-aware health and backend build information;
- region create, invite, join, leave and membership read;
- city claim and ownership read;
- publication lease acquire, resume, renew and release;
- regional presence and publication status;
- snapshot upload authorization, publication commit, head read and download authorization.

Mutations carry durable UUIDv4 action identity and canonical content. The Worker assigns a diagnostic trace ID to each HTTP attempt and returns it with errors; this trace has no idempotency or domain meaning. The Worker never accepts client-computed authorization, ownership or resource URLs.

[NETWORK.md](NETWORK.md#development-contract) owns HTTP compatibility policy, [wire numeric representation](NETWORK.md#wire-numeric-representation) and [validation/error limits](NETWORK.md#data-limits-and-abuse-boundaries).

The C# gateway maps stable machine error codes to typed application failures; it never interprets free-form message text. Shared protocol fixtures cover valid, invalid, boundary and canonicalization cases, including numeric limits, missing/null fields and unknown properties. Both language implementations must pass the same fixtures.

## Backend persistence

The backend is a single pnpm package and Cloudflare deployment. Its Worker entry point is stateless. Hono performs only HTTP concerns: routing, bounded parsing, authentication middleware, protocol validation and translation between HTTP responses and plain regional results. Hono request contexts do not enter regional rules or persistence code. Those framework-independent rules, the Durable Object class and a small `RegionStore` live together under `src/region` until their size demonstrates a need for more subdivision.

One region Durable Object owns a SQLite schema. Its persistence adapter issues SQL directly and binds every data value as a parameter; it does not concatenate request values into SQL and does not use an ORM or query builder. Static schema statements and closed internal identifier lists are the only places identifiers appear in SQL text.

Ordered migrations ship as application code with the deployment. On Durable Object initialization, the application reads its schema-version ledger and applies each missing migration once in order, transactionally where the platform operation permits. Requests are not handled against a partially migrated schema. Do not depend on a separately run operator migration step for per-region databases.

These application migrations change tables inside each region's SQLite database. They are separate from the Durable Object class-lifecycle declarations in Wrangler configuration; changing one does not substitute for updating the other.

Remote development data persists across deployments and follows those migrations. Deployment never wipes it automatically. When implementation needs a clean state, an explicit backend-owned development reset command must name the exact environment and region scope, use operator credentials that are unavailable to the game client, and emit an audit record. Add that command only when there is data to reset.

The schema stores:

- region metadata;
- membership, invite hashes and city claims;
- publication leases and deadline rows;
- processed actions and canonical request hashes;
- publication revisions, immutable object keys, checksums, sizes and current heads;
- structured non-sensitive city status summaries.

R2 stores immutable compressed visit-snapshot bytes and uncommitted pending-publication objects. The SQLite publication record stores only service-owned metadata such as object key, revision, byte checksum, compressed size and declared opaque format identifiers. Lifecycle cleanup may remove abandoned pending-publication data, but committed objects remain until a tested retention rule supersedes them.

The backend never interprets or simulates the city, never defines a partial backend copy of visit state and never treats a visit snapshot as authoritative economy state.

## Identity and trust

The invite-only MVP trusts the local simulation values contained in an owner's published snapshot. It protects ownership and prevents accidental duplication; it is not an anti-cheat economy.

Device-bound anonymous authentication is acceptable for the owner's disposable development regions. Public distribution, rankings, trading and valuable long-lived cities require recoverable identity, abuse controls and a separate trust design when that work is actually scheduled.

## Testing strategy

### Edit Mode

- Every rule and deterministic ordering listed in [SIMULATION_SPEC.md](SIMULATION_SPEC.md).
- ScriptableObject conversion, readable-ID uniqueness and missing-reference validation.
- Versioned JSON save round trips, adapter recovery and migrations from checked-in generations.
- Fixed city/traffic clocks at pause, normal and fast speed.
- Canonical snapshot projection and secret removal.
- Membership/publication outbox crash boundaries, response loss and offline save coalescing.
- Every shared protocol fixture against the hand-written C# implementation.

### Play Mode

- Camera, blueprint, very short construction, bulldozer and upgrade tools; upgrades retain occupants and existing operation until completion.
- Road/water connectivity, citywide power, persistent pollution, staffing, fire response and shared traffic overlays.
- Vehicle interpolation never alters simulation state.
- Visit mode exposes inspection but no editing or simulation advance.
- Development admin controls are absent from release configuration.

### Backend

- Every shared protocol fixture against the hand-written TypeScript implementation.
- Fast Vitest unit tests for framework-independent regional rules.
- Cloudflare's Vitest Worker integration for the HTTP boundary, real bindings, Durable Object isolation, SQLite storage and alarms. Node mocks are not the integration-test authority.
- Authentication, membership, invites, claims and publication-lease races.
- Idempotent action result and mismatched-content rejection.
- Restart recovery, ordered application migrations and deadline alarms; no effect-delivery tests before that system is scheduled.
- Parameter binding for all request-controlled SQLite values.
- Snapshot authorization, opaque-byte checksum, compressed size, pending publication, commit and access control.

### End to end

- Build a city, save it, publish it, lose a response, retry, and observe one head.
- Visit that snapshot while the owner is offline.
- Open and edit a region-bound city offline; reconnect and publish without stopping local simulation. Expired publication leases and stale-head errors never block local saves.
- Reject an incompatible or corrupt snapshot before scene construction.
- Crash at every local save and publication boundary and recover without duplicate cargo, credits, actions or heads.

## Performance and profiling

Record baseline hardware and create hashed canonical fixtures before enforcing frame, tick, routing or size budgets. Measure:

- city-tick p50/p95/max;
- traffic-step p50/p95/max by vehicle and intersection count;
- road-graph rebuild and route-query time;
- vehicle rendering and overlay cost;
- save, load, snapshot projection, compression and upload size;
- Worker latency, Durable Object time, SQLite operations and R2 operations.

## Builds and environments

- Local simulation tests run without Unity scenes.
- Development uses local Cloudflare emulation and one isolated remote development deployment with its own Durable Object namespace, R2 bucket, keys and data.
- Staging and production deployments and resources are not created until a release workflow requires them.
- Once backend code and its tests exist, a push to `main` that changes `backend` or `protocol` runs the relevant tests and automatically deploys the remote development environment only if they pass. Do not create an empty workflow before then.
- A remote-development game build should come from the same `main` revision as the deployed backend. A protocol change may create a short incompatibility window for an older local build; this is acceptable while the owner is the sole tester.
- Once regional networking exists, Unity Editor and development players select the latest remote development deployment by default. The development-only settings UI may redirect `HttpRegionGateway` to the local emulator when backend work needs interactive debugging. Committed environment profiles contain public endpoint and binding identifiers only; deployment secrets and user credentials are not Unity configuration.
- The resolved Git commit SHA is the build identity for both the game and backend. It is recorded in game saves and snapshots and exposed by developer diagnostics and the backend health response. A development build from modified tracked files displays `<sha>-dirty`; the clean SHA remains the base identity, and schema, content and rules versions remain independent integers.
- At the first player-build milestone, add one C# build entry point in `Zigrasite.Editor`. Unity menu commands and later batch-mode CI invoke the same code to select the two build scenes, target profile, scripting backend, development flags, Git identity and ignored `game/Builds/` destination. Do not maintain a separate shell implementation of the build policy.
- macOS validates all MVP traffic, oil, save and regional gates. Windows remains a target, but runtime validation is deferred until after the MVP; lack of Windows access does not block MVP completion.
- A macOS host may produce a Windows x64 Mono player, but that does not prove Windows runtime behavior. After the MVP, obtain Windows access and run the integrated gameplay, save and visit checks. A Windows IL2CPP build also requires a Windows build environment.

At Unity initialization, select and lock an exact Unity 6.3 LTS patch and compatible Unity packages. Commit Unity's project version and package lockfile. Select and lock Node.js, pnpm, TypeScript, Hono, Wrangler and backend test-package versions when backend implementation begins; commit its package lockfile and runtime declaration then. No version is selected only because it appears in an example scaffold.

Initialize `game` from the minimal Universal 3D template offered for that exact editor patch. Keep its required URP renderer and quality configuration, remove sample or tutorial content, and install only packages justified by current decisions. Do not hand-configure URP from an unrelated template merely to produce the same baseline.

## Decisions deferred until evidence exists

- Final Mono versus IL2CPP player-build policy after the staged macOS IL2CPP check.
- Initial numeric save schema, snapshot format, content manifest, simulation rules, traffic rules and backend schema versions, assigned when their first concrete artifacts are created.
- Exact footprint, map and cell dimensions.
- Simulation and traffic rates after profiling and visual tests.
- Additional road types, intersection controls and direct commuter consequences after the MVP.
- Save retention, snapshot cadence and size budgets.
- Meaningful research choices and content migrations; no research state or runtime centre in the MVP.
- Cloud recovery, multi-device editing and recoverable player accounts.
- Public-title or company-name changes after saves exist and the corresponding persistent-data-path migration.
- Mod support and any public content package format; neither is currently promised.
- Public-region trust, moderation and matchmaking.
- Cross-city action payloads, economy rules and general city-effect delivery.
