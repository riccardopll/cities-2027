# Network specification

## Purpose

The MVP network creates private two-city regions and publishes read-only city snapshots that the other owner can explore. It does not exchange electricity, workers, jobs, goods, money, services or simulation effects.

The architecture supports later regional actions through defined authorities, idempotency and ordered effects. Trade rules are undefined.

## Authority model

- The versioned local save is the only editable authority for roads, buildings, population, vehicles, inventories, research, money and city simulation.
- The per-region Durable Object owns membership, invites, city claims, active editor leases, publication heads, generic processed-action records and per-city effect sequences.
- R2 stores immutable compressed published city snapshots.
- A published snapshot is a read-only view artifact. It is not an editable cloud save and is not a recovery source.
- The Worker validates identity and requests, then routes region mutations to the correct object.

Only the owning client with a valid lease may edit a city history. Snapshot and regional records are read-only with respect to city simulation state.

## Deployed architecture

```text
Unity client
    |-- authoritative local city save
    |-- persistent regional action outbox
    |-- last applied city-effect sequence
    |
    | HTTPS / versioned JSON
    v
Cloudflare Worker
    |-- authentication and request validation
    |-- short-lived R2 upload/download authorization
    v
Region Durable Object
    |-- SQLite membership, claims, leases and publication heads
    |-- processed action ledger and ordered city effects
    |
    v
R2 immutable published snapshot blobs
```

Each private region routes to one SQLite-backed Durable Object. The MVP has no searchable public directory. D1 may be added later for discovery without becoming the authority for a region.

## Build stages

Networking begins only after the local city, traffic and oil gates pass.

1. **In-process gateway.** `LocalRegionGateway` and a durable test store exercise membership, claims, leases, action retries and publication-head rules.
2. **Local HTTP service.** Two executable clients use the real protocol against a local Worker and object implementation.
3. **Staging.** The same clients publish and visit snapshots through isolated Cloudflare resources and real authentication.

The in-process implementation follows the same authority and idempotency rules. It is a test double, not a separate game mode or simulated neighbour.

## Identity, membership and claims

- Current development may use device-bound anonymous Unity Authentication for the owner's disposable regions.
- Lost local credentials may orphan those development cities until recoverable identity and cloud recovery are designed.
- The Worker validates token signature, issuer, audience/project, time claims and player identity.
- A region is created with a client-generated opaque random region ID and permanent `createActionId`.
- An invite contains the opaque region ID and a high-entropy single-use secret. The object stores only its hash.
- Join and claim commands use permanent action IDs. Identical retries return the original result; ID reuse with different canonical content is rejected.
- The region has exactly two claimable city slots in the MVP and one owner per slot.

## Editing leases

One authenticated session may edit a city at a time.

- A city must connect, authenticate, reconcile regional state and acquire a lease before local simulation or editing begins.
- The lease token is saved installation-locally before the city starts ticking so the same session can resume after a crash.
- Renew every 30 real seconds against an initial 120-second lease.
- After two consecutive renewal failures, pause simulation and editing before expiry. Camera and menus may remain available.
- A different session cannot replace a valid lease.
- The MVP does not merge copied local saves or support simultaneous devices.

Leases establish the single-writer rule for snapshot publication and future regional actions.

## Snapshot publication

### When publication is requested

The client may publish only after a durable compatible local save. It schedules publication:

- after qualifying durable saves;
- during a clean exit; and
- at a configurable fixed batch cadence while the city is open.

Saving locally must never wait for the network. Clean exit makes a best-effort publication, but correctness does not depend on it completing.

### Snapshot contents

A visit snapshot is a data-only, read-only representation sufficient to reconstruct the visible city and inspectors:

```text
snapshotSchemaVersion
simulationRulesVersion
regionId, cityId and owner-safe display metadata
localSaveRevision and publicationRevision
publishedAt
map and content-definition references
roads, blueprints, buildings and ruins
occupancy, utilities, fire and research state
traffic assignments and a stable visit-time vehicle presentation state
oil deposits, inventories, facilities and declared production state
non-sensitive city statistics and overlay inputs
```

It contains no authentication token, lease token, invite secret, local file path, unpublished action, debug command history or private save-generation metadata.

Visitors do not continue the city simulation. Moving cars or facility animation in visit mode may loop or interpolate presentation data, but cannot advance authoritative traffic, production, risk, money or clocks.

### Atomic publish flow

1. Finish and verify the local save.
2. Build canonical snapshot bytes and compute uncompressed and compressed SHA-256 checksums.
3. Create a permanent publication action ID and save it in the local outbox.
4. Ask the region for a short-lived upload authorization tied to city, action, size and checksum.
5. Upload to a unique immutable staging key.
6. Ask the region object to verify the object and atomically commit the next publication head.
7. Store the canonical result under the action ID and return it on identical retries.
8. Remove the local outbox action only after the committed result is durably saved.

A blob is never visible as the city head before verification and object existence. A failed upload or abandoned staging object cannot replace the previous valid head. Lifecycle rules remove uncommitted staging objects later.

Publication revisions increase monotonically. An older local save cannot replace a head produced from a newer save revision unless an explicit future recovery protocol authorizes it.

## Visiting a city

1. Authenticate and open the private region.
2. Read membership, city owner, online/offline state and current publication head.
3. Request a short-lived download authorization for the immutable blob.
4. Verify compressed checksum, decompress within strict limits and verify the canonical checksum.
5. Validate snapshot schema, simulation rules, content manifest and size limits.
6. Load visit mode with all edit, save, admin and simulation-advance commands disabled.

The UI shows the snapshot revision and last-published time. An incompatible snapshot produces an error and is not loaded. Visits read the published artifact while the owner is offline.

## Generic future action foundation

The MVP implements and tests the following infrastructure even though it has no cross-city gameplay action types:

- A persistent client outbox with permanent action IDs and monotonic client action sequence.
- Canonical request hashing performed by the service.
- One processed-action record with one permanent result per ID.
- Per-city ordered effect sequences with gap-free application, durable local save before acknowledgement and safe replay after rollback.
- Region revisions that never move backward.
- Transactional mutation of region state, action result and generated effects.

Create, join, claim and snapshot publication exercise idempotent actions. Lease and publication status exercise ordered reconciliation. Later trade may reuse these mechanics, but its prices, inventory rules, time model and authority still require separate design.

## Synchronization

While editing, the client synchronizes and renews every 30 real seconds. A sync request includes:

- region, city, client-session and lease data;
- compatible protocol and simulation versions;
- current local save and publication revisions;
- last applied city-effect sequence;
- pending actions in stable client order; and
- current non-sensitive status summary.

For one request, the region object:

1. validates authentication, membership, versions, sizes and canonical hashes;
2. processes due deadlines idempotently;
3. validates or renews the lease;
4. processes pending actions in stable order;
5. commits action results, effects and region revision in one transaction; and
6. returns the current region snapshot plus the next page of effects.

The client applies effects without gaps, saves the new effect position, then acknowledges. No MVP effect changes city gameplay, but the rule is established before later regional mechanics need it.

## Data limits and abuse boundaries

- Every route has explicit body, decompressed-body, field-count, collection-count and string-length limits.
- Snapshot uploads have declared compressed and uncompressed size limits and checksum binding.
- Decompression is streaming or bounded to prevent archive bombs.
- Region and city authorization is checked on every metadata and blob request.
- Rate limits protect create, join, publish, download and synchronization endpoints.
- Published cities in the invite-only MVP trust player-authored local simulation values. Public discovery and competitive ranking require a separate anti-cheat and moderation design.

## Deadlines and alarms

The region stores all deadlines in one table because a Durable Object has one active alarm. MVP deadlines include invite expiry, lease expiry and cleanup of uncommitted snapshot staging data. Processing is idempotent and re-arms unfinished work after failure.

Wall time never advances a city's population, vehicles, production, fire, money or research.

## Correctness invariants

- One city has one owner and at most one valid editing lease.
- One action ID has one canonical request and permanent result.
- Region, publication and city-effect revisions never move backward.
- A regional mutation is durable before success is returned.
- A client saves an effect before acknowledging it.
- A visit snapshot is immutable and never editable.
- A publication head never references an unverified or missing blob.
- A stale local save cannot silently replace a newer published head.
- Visiting never advances or mutates the owner city's simulation.
- Local saves never contain access tokens, invite secrets or storage credentials.
- The backend contains no trade actions or trade schemas in the MVP.

## Explicit MVP exclusions

- Cross-city utilities, workers, jobs, vehicles, goods, money, services, pollution and projects.
- Live viewing of another player's current simulation.
- Shared editing, lockstep, peer-to-peer or player-hosted networking.
- Public discovery, matchmaking, ranking, moderation and competitive authority.
- Cloud save recovery, cross-device editing and save merging.
- Full city simulation on the server.
- WebSockets, queues and cross-region analytics unless measured need justifies them.

## Required tests

- Duplicate create, join, claim and publication attempts return one result.
- Concurrent city claims and leases have one winner.
- Lost responses, process restarts and delayed acknowledgements do not create duplicate heads or effects.
- A blob upload without committed metadata is never visitable.
- A committed head always resolves to a checksum-valid immutable blob.
- Unauthorized players cannot read metadata or snapshots from another private region.
- Schema, rules, content or size incompatibility fails before Unity objects are created.
- Visitors can inspect an offline owner's latest snapshot but cannot issue edit commands.
- Fixed-cadence publication coalesces redundant saves and never blocks local saving.

## Decisions for later evidence

- Exact fixed publication cadence and snapshot size budget.
- Region location and data-residency policy before public distribution.
- Recoverable player identity and true cloud-save recovery.
- Public discovery, moderation and anti-cheat requirements.
- Notification transport after polling is measured.
- Every cross-city gameplay rule.
