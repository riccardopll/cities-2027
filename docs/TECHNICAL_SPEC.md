# Technical specification

## Purpose

This document defines the code boundaries, persistence, protocol and verification strategy for the current game direction. Player-facing rules live in [GAMEPLAY.md](GAMEPLAY.md); exact local ordering lives in [SIMULATION_SPEC.md](SIMULATION_SPEC.md); deployed regional behaviour lives in [NETWORK.md](NETWORK.md).

## System overview

```text
Unity presentation
    -> application commands and queries
        -> plain-C# deterministic city simulation
        -> plain-C# deterministic traffic simulation
        -> save/publish coordinators
            -> local immutable save generations
            -> IRegionGateway
                -> LocalRegionGateway for tests
                -> HttpRegionGateway for deployed service

Cloudflare Worker
    -> per-region SQLite Durable Object
    -> immutable R2 visit snapshots
```

The local save owns the editable city. The region owns identity, membership, claims, leases, publication heads and future-safe action/effect ordering. A published city is a read-only projection, never a second editable authority.

## Decisions

- Unity 6.3 LTS client for Windows and macOS.
- C# simulation assemblies without dependencies on `UnityEngine`.
- Orthogonal authored grid with fixed rectangular building footprints.
- Integer gameplay values and explicit stable ordering.
- Separate fixed city and traffic update rates.
- Local authoritative saves with immutable generations and migrations.
- Cloudflare Worker, one SQLite-backed Durable Object per region and R2 for immutable visit snapshots.
- Private two-city MVP regions with one active editor lease per city.
- JSON/HTTPS protocol with strict versions and limits until measurement justifies another encoding.
- Persistent action IDs, canonical hashes, generic processed-action ledger and ordered effects from the first deployed foundation.
- No ECS, Burst, Jobs, binary protocol, WebSockets or peer-to-peer networking without a measured need.

## Repository layout

```text
/
|-- AGENTS.md
|-- docs/
|-- client/
|   |-- Assets/
|   |   |-- Game/
|   |   |   |-- Simulation/
|   |   |   |-- Traffic/
|   |   |   |-- Application/
|   |   |   |-- Infrastructure/
|   |   |   `-- Presentation/
|   |   `-- Tests/
|   |       |-- EditMode/
|   |       `-- PlayMode/
|   `-- Packages/
|-- backend/
|   |-- src/
|   |   |-- worker/
|   |   `-- region-object/
|   |-- migrations/
|   `-- test/
|-- protocol/
|   |-- schemas/
|   |-- fixtures/
|   `-- generated/
|-- tools/
|   |-- fixtures/
|   |-- migration-checks/
|   `-- load-tests/
`-- .github/workflows/
```

Shared protocol fixtures are language-neutral.

## Client dependency boundaries

### Simulation

Owns deterministic city state and rules:

- grid, terrain, deposits, streets, blueprints, buildings and stable IDs;
- construction, upgrades, occupancy, jobs, utilities, fire, money and research;
- oil inventories, recipes, reservations and export accounting;
- serializable state and domain events.

It does not reference Unity objects, HTTP, authentication, file paths or rendered time.

### Traffic

Owns deterministic routes and moving trips:

- road graph projection and dirty rebuilds;
- home/job assignments and commute schedules;
- production dispatch reservations;
- vehicle routes, fixed-step positions, queues, arrivals and congestion metrics;
- serializable or deterministically reconstructable traffic state.

Traffic may query immutable city facts through narrow interfaces. It changes city inventory only through validated arrival events at fixed boundaries.

### Application

Coordinates use cases:

- validated player and admin commands;
- pause, normal and fast scheduling;
- fixed-step boundary application;
- save, load, migration and recovery flow;
- region create/join/claim and lease lifecycle;
- snapshot building, publication, download and visit-mode entry;
- durable outbox and city-effect application.

### Infrastructure

Implements:

- atomic local files and immutable save generations;
- compression, hashes and canonical serialization;
- Unity Authentication adapter;
- `LocalRegionGateway` and `HttpRegionGateway`;
- retry, timeout, backoff and clock adapters;
- structured logs and metrics.

### Presentation

Owns Unity scenes, GameObjects, cameras, tools, overlays, inspectors, audio and interpolated visuals. It issues commands and reads snapshots. It never mutates simulation collections directly or decides whether cargo arrived.

## Content definitions

Building, recipe, vehicle and tuning definitions are data assets converted into immutable simulation definitions at startup. Saved entities reference stable definition IDs and store any state that cannot safely be re-derived.

Required definition categories include:

- homes, shops, ordinary factories and upgrades;
- streets, generators, water towers, fire stations and research centres;
- oil deposits, drills, refineries, trucks and cargo ports;
- footprints, access edges, costs, capacities, upkeep, tax, nuisance, contamination, coverage and risk;
- recipes, storage, cargo sizes, processing durations and export values.

Content IDs are never derived from display names or asset paths. Removing or changing a definition requires a migration or an explicit incompatible test-save reset.

## Fixed-step scheduling

The application owns an accumulator for fixed city ticks and fixed traffic steps. Render frames consume immutable presentation snapshots and interpolate vehicle transforms.

- Pause processes UI, saving and networking but advances no simulation clocks.
- Normal and fast speed change how many complete fixed steps are processed.
- A maximum work budget prevents a slow frame from causing an unbounded catch-up spiral; hitting it produces a metric and visible development warning.
- A save is captured only at a complete declared boundary.
- Asynchronous results enter through the between-step pump and are recorded for replay.

Traffic results that transfer production cargo are emitted as immutable arrival events and applied exactly once at the next permitted city boundary.

## Save system

### Envelope

Every save contains:

```text
magic and saveSchemaVersion
gameBuild and contentManifestVersion
simulationRulesVersion and trafficRulesVersion
cityId, region reference and city seed
localSaveRevision
cityTick, trafficStep and speed state
next stable entity, vehicle and command IDs
map, roads, blueprints, buildings and ruins
occupancy, job assignments and utility state
fire risk and coverage state
research points and balances
oil deposits, inventories, recipes and reservations
vehicle routes, queues and cargo or sufficient reconstruction state
regional publication revision
last applied city-effect sequence
pending regional action outbox
payload length and SHA-256 checksum
```

Runtime references, GameObjects, access tokens, lease tokens, invite secrets and storage credentials are forbidden.

### Immutable generations

Write a save to a unique temporary generation, flush it, verify length and checksum, then atomically advance a tiny head record. Keep several known-good generations and scan them newest-first when the head is missing or corrupt.

Never overwrite the only valid city file. A migration reads an old generation and writes a new generation; it does not alter the source.

Autosave cadence is tuning and must not stall fixed-step processing. Important construction, cargo and regional boundaries may request an immediate save through the coordinator.

### Outbox and ordered effects

Before its first send, a regional mutation receives a UUID action ID, canonical payload and monotonic client sequence and is saved in the outbox. Retry the same bytes until the service returns the stored result.

Apply incoming city effects in sequence without gaps. Save the resulting local state and new sequence before acknowledging. The MVP has few effect types, but publication and lease reconciliation prove the mechanism before future cross-city actions depend on it.

## Visit snapshot projection

The snapshot builder reads one verified local save boundary and emits only data required for read-only reconstruction. It removes secrets, admin history, unpublished actions and recovery internals.

Canonical serialization must provide:

- stable field and collection ordering;
- explicit schema, simulation, traffic and content versions;
- compressed and uncompressed length limits;
- SHA-256 for canonical and compressed bytes;
- deterministic output for identical source state.

Visit mode loads into separate read-only application state. Edit tools, simulation scheduling, local saving, admin commands and regional mutation commands are not constructed for a visited city.

## Regional protocol

### Route groups

The exact HTTP paths may evolve, but the protocol requires route groups for:

- authentication-aware health and protocol negotiation;
- region create, invite, join and membership read;
- city claim and ownership read;
- lease acquire, resume, renew and release;
- synchronized generic actions and ordered effects;
- snapshot upload authorization, publication commit, head read and download authorization.

Every request carries an explicit protocol version and bounded request ID. Mutations carry action identity and canonical content. The Worker never accepts client-computed authorization, ownership or resource URLs.

### Initial limits

Set and test explicit limits for request bytes, decompressed bytes, strings, arrays, action batches, effects per page, snapshot sizes, region count per account and request rates. Keep limits in shared fixtures so C# and TypeScript reject the same boundary cases.

Use decimal strings for counters that may exceed JavaScript's safe-integer range, or prove through shared tests that bounded integers cannot exceed it.

## Backend persistence

The Worker is stateless. One region Durable Object owns a SQLite schema with idempotent migrations for:

- region metadata and protocol versions;
- membership, invite hashes and city claims;
- leases and deadline rows;
- processed actions and canonical request hashes;
- per-city gap-free effect sequences and acknowledgements;
- publication revisions, immutable object keys, checksums, sizes and current heads;
- structured non-sensitive city status summaries.

R2 stores immutable compressed visit snapshots and uncommitted staging objects. Lifecycle cleanup may remove abandoned staging data, but committed objects remain until a tested retention rule supersedes them.

The backend never simulates the city and never treats a visit snapshot as authoritative economy state.

## Identity and trust

The invite-only MVP trusts the local simulation values contained in an owner's published snapshot. It protects ownership and prevents accidental duplication; it is not an anti-cheat economy.

Device-bound anonymous authentication is acceptable for the owner's disposable development regions. Public distribution, rankings, trading and valuable long-lived cities require recoverable identity, abuse controls and a separate trust design when that work is actually scheduled.

## Testing strategy

### Edit Mode

- Every rule and deterministic ordering listed in [SIMULATION_SPEC.md](SIMULATION_SPEC.md).
- Save round trips and migrations from checked-in generations.
- Fixed city/traffic clocks at pause, normal and fast speed.
- Canonical snapshot projection and secret removal.
- Outbox and ordered-effect crash boundaries.

### Play Mode

- Camera, blueprint, construction, bulldozer and upgrade tools.
- Road, utility, contamination, fire and congestion overlays.
- Vehicle interpolation never alters simulation state.
- Visit mode exposes inspection but no editing or simulation advance.
- Development admin controls are absent from release configuration.

### Backend

- Authentication, membership, invites, claims and lease races.
- Idempotent action result and mismatched-content rejection.
- Gap-free effects, restart recovery, migrations and deadline alarms.
- Snapshot authorization, checksum, size, staging, commit and access control.

### End to end

- Build a city, save it, publish it, lose a response, retry, and observe one head.
- Visit that snapshot while the owner is offline.
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
- Development, staging and production use distinct backend resources, keys and data.
- macOS is sufficient for the scale and kernel spikes.
- A Windows smoke build is required at the local-city presentation gate.
- Both platforms run later traffic, oil, save and regional checklists.

## Decisions deferred until evidence exists

- Exact footprint, map and cell dimensions.
- Simulation and traffic rates after profiling and visual tests.
- Intersection, lane and congestion coupling beyond the MVP score.
- Save retention, snapshot cadence and size budgets.
- Meaningful research choices and content migrations.
- Cloud recovery, multi-device editing and recoverable player accounts.
- Public-region trust, moderation and matchmaking.
- Cross-city action payloads and economy rules.
