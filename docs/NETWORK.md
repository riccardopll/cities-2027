# Network specification

## Purpose

The MVP network creates private two-city regions and publishes read-only city snapshots that the other owner can explore. It does not exchange electricity, workers, jobs, goods, money, services or simulation effects.

The architecture retains local city authority, regional ownership and reliable retries for membership and publication. Trades between cities and general city-effect delivery are deferred until a gameplay exchange is defined. The MVP has no effect stream, effect acknowledgements or trade transaction framework.

## Authority model

- The versioned local save is the only editable authority for roads, buildings, population, vehicles, inventories, ground pollution, money and city simulation.
- The per-region Durable Object owns membership, invites, city claims, publication leases, publication heads and processed-action records for implemented regional operations.
- R2 stores immutable compressed published city snapshots.
- A published snapshot is a read-only view artifact. It is not an editable cloud save and is not a recovery source.
- One Cloudflare Worker deployment contains the HTTP handler and the Region Durable Object module. The HTTP handler validates identity and requests, then routes region mutations to the correct object.

Local editing and simulation require only the exclusive local save-writer lock, including for region-bound cities. A publication lease authorizes changes to the online snapshot head; it never authorizes or pauses local play. Snapshot and regional records cannot mutate city simulation state. Editing copied saves on multiple devices and merging histories remain unsupported.

## Deployed architecture

```text
Unity client
    |-- authoritative local city save
    |-- persistent regional action outbox
    |
    | HTTPS / strictly validated JSON
    v
Cloudflare Worker deployment
    |-- HTTP handler module
    |   |-- authentication and request validation
    |   `-- short-lived R2 upload/download authorization
    |-- Region Durable Object module
    |   |-- SQLite membership, claims, leases and publication heads
    |   `-- stored results for membership and publication retries
    |
    v
R2 immutable published snapshot blobs
```

The HTTP handler and Durable Object class ship as one deployable Worker, not as separately deployed services. Each private region routes to one SQLite-backed Durable Object instance. The MVP has no searchable public directory. D1 may be added later for discovery without becoming the authority for a region.

## Build stages

Networking begins only after the local city, traffic and oil gates pass.

1. **Client boundary tests.** Controlled `IRegionGateway` responses exercise client success, failure, retries and publication status. They contain no durable region store or second implementation of regional rules.
2. **Local Cloudflare emulator.** Two executable clients use the real HTTP protocol against the same Worker entry point, Durable Object module and bindings used by the deployed build.
3. **Remote development environment.** The same clients publish and visit snapshots through one isolated set of Cloudflare development resources with real authentication.

Ownership, membership and publication rules are implemented once in the real TypeScript backend. Test its behavior against the local Worker runtime. Client test doubles supply selected responses only; they are not alternate services, durable backends or simulated neighbours.

Backend rule tests use pure TypeScript where no runtime service is involved. Integration tests use Cloudflare's Vitest Worker runtime with real bindings, Durable Object isolation, SQLite and alarms; mocked Node implementations are not treated as proof of deployed behavior.

Staging and production environments are not provisioned during solo development. They are added only when distribution or release work creates a concrete need for them.

When backend implementation exists, pushes to `main` that change `backend` or `protocol` run the relevant tests and automatically update the remote development environment only after those tests pass. The workflow is added with the code and tests, not as empty scaffolding. Remote integration testing uses a game build from the same `main` revision. A protocol change may temporarily make an older local build incompatible with remote development; that is acceptable while the owner is the only tester.

Remote development data survives deployments and follows the numbered application schema migrations. Deployment does not reset Durable Object or R2 state. Any future development reset is an explicit backend operator command scoped to a named environment and region, protected by credentials unavailable to the game client, and recorded in an audit log.

## Development contract

During solo development, the client, local emulator and remote development environment implement one current, unversioned HTTP contract. The client and Worker change together. A stale client is unsupported after remote development updates and may receive ordinary route or validation errors. The service has no version header, negotiation, translation or dedicated protocol-incompatibility response.

Introduce explicit network versioning before external distribution or when different client contracts must run concurrently. Snapshot schema, simulation-rules and content-manifest versions remain separate now because stored snapshots can outlive a network deployment.

## Identity, membership and claims

- Current development may use device-bound anonymous Unity Authentication for the owner's disposable regions.
- Lost local credentials may orphan those development cities until recoverable identity and cloud recovery are designed.
- The Worker validates token signature, issuer, audience/project, time claims and player identity.
- A region is created with a client-generated canonical UUIDv4 region ID and permanent UUIDv4 `createActionId`.
- A city receives a canonical random UUIDv4 before its first local save. Claiming atomically binds that client-created ID to one region slot and owner; a conflicting prior claim is rejected rather than remapping the local city.
- An invite contains the opaque region ID and a high-entropy single-use secret. The object stores only its hash.
- Join and claim commands use permanent UUIDv4 action IDs. Identical retries return the original result; ID reuse with different canonical content is rejected.
- Region, city and durable action IDs use lowercase hyphenated UUIDv4 strings, with a fresh value for every region, city and action. They are not player-facing and do not receive a Base58 encoding. In-city entity counters and invite secrets retain separate formats. The backend creates a diagnostic trace ID for each HTTP attempt and includes it with errors; clients do not supply a separate request ID.
- The region has exactly two claimable city slots in the MVP and one owner per slot.

## Offline play and publication leases

Standalone and region-bound cities can open, simulate, edit and save without a network connection. A local exclusive writer lock prevents two processes from editing the same local city directory.

One authenticated session at a time may publish a city's snapshot:

- Connect, authenticate, read regional ownership and the current head, and acquire a publication lease before beginning a new publication.
- A session may retain its lease token installation-locally for crash recovery; it is never included in a city save or visit snapshot.
- While publishing is enabled online, renew every 30 real seconds against an initial 120-second lease. These values remain tuning data.
- Renewal failure or lease expiry suspends publication attempts that require a valid lease. It never pauses local editing, simulation or saving.
- A different session cannot replace a valid publication lease. Report publication conflicts without blocking the local city.
- On reconnect, resolve pending action results and verify the visible head before publishing again. A stale or conflicting local history cannot silently replace a newer head; the error affects publication only.

Membership changes and visits require a connection. Offline edits need no regional approval. Offline saves coalesce into the newest eligible publication request; do not enqueue a snapshot for every offline autosave. Preserve the identity and bytes of actions already sent until their outcome is resolved.

The MVP does not support multiple-device editing or save merging. A publication lease serializes online writes; it cannot establish that two copied local saves share the same history. Do not describe it as a global editing lock. Future trade must define its own offline and transaction rules without changing MVP offline play by accident.

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
occupancy, staffing, utilities, ground pollution, fire risk and response state
traffic assignments and a stable visit-time vehicle presentation state
oil deposits, inventories, recorded resource losses, facilities and declared production state
non-sensitive city statistics and overlay inputs
```

It contains no authentication token, lease token, invite secret, local file path, unpublished action, debug command history or private save-generation metadata.

Visitors do not continue the city simulation. Moving cars or facility animation in visit mode may loop or interpolate presentation data, but cannot advance authoritative traffic, production, risk, money or clocks.

The complete serialized and compressed visit snapshot is opaque to the backend. The service does not deserialize, decompress or validate its gameplay fields. It validates the separate publication metadata, including identifiers, versions, revisions, timestamps, declared compressed and uncompressed byte counts, and checksum formats. Before committing a publication head, it also confirms that the immutable R2 object exists and that its stored compressed byte count and checksum match the bound publication metadata. The visiting client performs bounded decompression, verifies the uncompressed checksum and validates the snapshot contents.

### Atomic publish flow

1. Finish and verify the local save.
2. Build canonical snapshot bytes and compute uncompressed and compressed SHA-256 checksums.
3. Create a permanent publication action ID and save it in the local outbox.
4. Ask the region for a short-lived upload authorization tied to city, action, size and checksum.
5. Upload to a unique immutable pending-publication key.
6. Ask the region object to validate the publication metadata, verify the opaque object's compressed size and checksum, and atomically commit the next publication head.
7. Store the canonical result under the action ID and return it on identical retries.
8. Remove the local outbox action only after the committed result is durably saved.

A blob is never visible as the city head before verification and object existence. A failed upload or abandoned pending-publication object cannot replace the previous valid head. Lifecycle rules remove uncommitted pending-publication objects later.

Publication revisions increase monotonically. An older local save cannot replace a head produced from a newer save revision unless an explicit future recovery protocol authorizes it.

## Visiting a city

1. Authenticate and open the private region.
2. Read membership, city owner, online/offline state and current publication head.
3. Request a short-lived download authorization for the immutable blob.
4. Verify compressed checksum, decompress within strict limits and verify the canonical checksum.
5. Validate snapshot schema, simulation rules, content manifest and size limits.
6. Load visit mode with all edit, save, admin and simulation-advance commands disabled.

The UI shows the snapshot revision and last-published time. An incompatible snapshot produces an error and is not loaded. Visits read the published artifact while the owner is offline.

## Reliable membership and publication actions

Implement reliable retries for actual MVP operations: create region, invite, join, claim, leave and publication, plus lease mutations where required by their contracts.

- Save an operation's permanent action ID and canonical payload before its first send in a persistent local outbox.
- Hash the validated canonical request on the service.
- Store one result per action ID. Identical retries return that result; reuse with different content fails.
- Commit a regional mutation and its action result transactionally before returning success.
- Persist the resolved local operation status before removing its outbox entry.
- Keep region and publication revisions monotonic.

This outbox records pending regional requests; it does not apply trades to city money or inventory. No generic client action sequence, per-city effect stream, effect acknowledgement or trade replay system is implemented in the MVP.

## Online status and reconnection

While connected to a region, poll membership, owner presence and publication heads at an initial 30-real-second cadence. Renew publication leases separately or within the same bounded request where appropriate. The exact routes are defined with the first client/backend implementation.

Responses describe regional state and the results of requested membership or publication operations. They contain no ordered city effects. Validate ownership and head revisions before a new publication. Resolve lost responses using the original action IDs.

On disconnect, retain the last known regional status, show that online features are unavailable and keep local play running. Owner online/offline status means the service has recently heard from that session; it cannot report whether someone is playing locally without a connection. Snapshot publication time remains visible independently of presence.

## Wire numeric representation

All current revision and sequence counters use JSON numbers and must be non-negative integers no greater than `9007199254740991`, JavaScript's IEEE-754 safe-integer ceiling. C# can represent larger signed integers, but the client enforces the shared limit because the Cloudflare side cannot safely preserve larger JSON numbers. This applies to local-save, publication and region revisions, and to any other network counter. Clients and the service reject negative, fractional or unsafe values. An increment that would exceed the bound fails without committing a mutation; counters never wrap or lose precision.

Any future field that needs the full signed 64-bit range uses a canonical base-10 decimal JSON string bounded from `-9223372036854775808` through `9223372036854775807`. Zero is `"0"`; other values have no leading zero or leading plus sign. A non-negative counter additionally rejects negative strings. Introducing such a field or changing an existing field's representation requires a coordinated contract change; the current contract never accepts both number and string forms for one field.

## Data limits and abuse boundaries

- Every API failure uses one bounded JSON envelope with a stable lowercase machine code, safe message, backend trace ID and optional safe details such as a validation field path. HTTP status codes retain their broad meaning. Clients branch on the machine code, never the message; responses expose no stack, token, invite secret or SQL detail.
- Every route has explicit body, decompressed-body, field-count, collection-count and string-length limits. Set region-count limits per account and bounds for implemented operation payloads. Keep these limits in shared protocol fixtures so C# and TypeScript enforce the same boundaries.
- Typed request and response objects reject unknown JSON properties. Missing and `null` are distinct unless a field explicitly permits both, and the complete mutation DTO is validated before regional state is touched. Duplicate-property detection is not required; action hashes use the canonical validated value rather than ambiguous raw JSON.
- Snapshot uploads have declared compressed and uncompressed size limits and checksum binding.
- Snapshot publication metadata has explicit field and value limits; the backend verifies opaque blob length and checksum without parsing snapshot contents.
- Decompression is streaming or bounded to prevent archive bombs.
- Region and city authorization is checked on every metadata and blob request.
- Rate limits protect create, join, publish, download and regional-status endpoints.
- Published cities in the invite-only MVP trust player-authored local simulation values. Public discovery and competitive ranking require a separate anti-cheat and moderation design.

## Deadlines and alarms

The region stores all deadlines in one table because a Durable Object has one active alarm. MVP deadlines include invite expiry, lease expiry and cleanup of uncommitted pending-publication snapshot data. Processing is idempotent and re-arms unfinished work after failure.

Wall time never advances a closed city's population, vehicles, production, ground pollution, fire or money. An open city advances on its local fixed clocks whether online or offline.

## Correctness invariants

- One city has one regional owner and at most one valid publication lease. Local editing uses its own filesystem writer lock.
- One action ID has one canonical request and permanent result.
- Region and publication revisions never move backward.
- A regional mutation is durable before success is returned.
- Network failure never pauses local simulation, editing or saving.
- A visit snapshot is immutable and never editable.
- A publication head never references an unverified or missing blob.
- A stale local save cannot silently replace a newer published head.
- Visiting never advances or mutates the owner city's simulation.
- Local saves never contain access tokens, invite secrets or storage credentials.
- The backend contains no trade actions, trade schemas or general city-effect delivery in the MVP.

## Explicit MVP exclusions

- Cross-city utilities, workers, jobs, vehicles, goods, money, services, pollution and projects.
- General cross-city transaction delivery, effect sequences and effect acknowledgements.
- Live viewing of another player's current simulation.
- Shared editing, lockstep, peer-to-peer or player-hosted networking.
- Public discovery, matchmaking, ranking, moderation and competitive authority.
- Cloud save recovery, cross-device editing and save merging.
- Full city simulation on the server.
- WebSockets, queues and cross-region analytics unless measured need justifies them.

## Required tests

- Duplicate create, join, claim and publication attempts return one result.
- Concurrent city claims and publication leases have one winner; a losing publication attempt does not stop local play.
- Lost responses and process restarts do not duplicate membership actions or publication heads.
- A blob upload without committed metadata is never visitable.
- A committed head always resolves to a checksum-valid immutable blob.
- Unauthorized players cannot read metadata or snapshots from another private region.
- Schema, rules, content or size incompatibility fails before Unity objects are created.
- Visitors can inspect an offline owner's latest snapshot but cannot issue edit commands.
- Fixed-cadence publication coalesces redundant saves and never blocks local saving.
- Disconnect before opening or during play: both standalone and region-bound cities remain editable and saveable.
- Reconnect after offline edits: resolve pending publications, reject stale heads and publish a compatible newer save without restarting city simulation.
- Unsafe, fractional and overflowing protocol counters are rejected without precision loss or partial mutation.

## Decisions for later evidence

- Exact fixed publication cadence and snapshot size budget.
- Region location and data-residency policy before public distribution.
- Recoverable player identity and true cloud-save recovery.
- Public discovery, moderation and anti-cheat requirements.
- Notification transport after polling is measured.
- Every cross-city gameplay rule.
