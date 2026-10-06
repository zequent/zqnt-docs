# Edge SDK — Connector API Reference

> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Connector reference](edge-sdk-connector-reference-1.3.md).

Method reference for `ConnectorService`, the adapter's access to the platform's system of record:
pairing, asset records, payloads, media files and the Skill Registry.

All methods return a `CompletableFuture`. A call that fails in transport is retried (up to 5
attempts) before the future completes exceptionally. When the platform answers with an error, the
asset, payload and scheduler methods log it and resolve to `null` (or `false` for a delete) rather
than throwing.

An adapter calls with its edge credential (`ZQNT_EDGE_TOKEN`). Creating, changing and deleting
schedules is administration and is refused (`PERMISSION_DENIED`); everything else on this page is
allowed.

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `getAssetBySn(sn)` | `AssetDTO` | Get asset by serial number |
| `getAssetById(id)` | `AssetDTO` | Get asset by ID |
| `getSubAssetBySn(sn)` | `SubAssetDTO` | Get sub-asset by serial number |
| `ensureAsset(AssetDTO, claimCode)` | `AssetDTO` | Look the asset's serial number up, and redeem `claimCode` for it only if it is unknown. With no code it reports what it found and creates nothing |
| `redeemAssetClaim(code, AssetDTO)` | `AssetDTO` | Trade a one-time claim code for an asset, and return it |
| `describeAssetClaim(code)` | `String` | The name of the organization a claim code would provision into, without spending the code |
| `updateAsset(id, AssetDTO)` | `AssetDTO` | Update an existing asset |
| `deRegisterAsset(id)` | `Boolean` | Delete an asset record |
| `registerAsset(AssetDTO)` | `AssetDTO` | **Deprecated** — use `ensureAsset` |

**Pairing.** Call `ensureAsset` at startup. Because a claim is single-use, the lookup comes first:
after the first successful pairing there is nothing left to redeem, and the adapter would otherwise
log a refusal on every restart. Without a claim code, `ensureAsset` creates nothing — the normal
state for devices provisioned in the Admin Console.

`redeemAssetClaim` is the only `ConnectorService` call an adapter makes with no platform identity:
the code *is* the credential. The organization the created asset lands in comes from the claim,
never from the `AssetDTO` you pass (its organization field is ignored). A refused code completes
with `null`, and every refusal looks the same — unknown, expired, revoked, exhausted, or not valid
for this kind of device — so the call cannot be used to discover which codes exist.

`describeAssetClaim` is for the moment before anything is created: a device can show its operator
which organization it is about to join, and wait for them to confirm, without consuming the code.
It returns the organization's name and nothing else, and `null` for every unusable code alike.

`registerAsset` is insert-only: it fails for a serial number the platform already knows, and
otherwise creates an asset with no organization, which no organization can see and nobody can move
afterwards. It remains only for the DJI adapter's deprecated organization-id binding.

## Asset payloads

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsertAssetPayload(assetSn, subAssetSn, AssetPayloadDTO)` | `AssetPayloadDTO` | Create or update an asset payload |

## Media files

| Method | Returns | Purpose |
| --- | --- | --- |
| `registerMediaFile(RegisterMediaFileRequest)` | `MediaFileProtoDTO` | Register a file the device uploaded to the platform's inbox bucket |

The platform attributes the file to the execution that was running on the asset, moves it under its
organization and Application folder, and records it. Registration is idempotent per source object
key, so a retried report is harmless. The request's `base` is filled in when you leave it unset.

## Schedules

| Method | Returns | Purpose |
| --- | --- | --- |
| `getSchedulerById(id)` | `SchedulerDTO` | Read a schedule |
| `createScheduler` / `updateScheduler` / `deleteScheduler` | — | Refused for an edge credential: schedules are managed in the Admin Console |

`SchedulerDTO` (`com.zqnt.utils.missionautonomy.domains`, shared with the Client SDK) describes what
a schedule runs:

| Field | Type | Notes |
| --- | --- | --- |
| `name`, `cronExpression`, `clientTimeZone`, `active`, `type` | | When and whether it fires |
| `assetSn` | `String` | The asset it runs on. Optional for an Application schedule, which then picks an asset at firing time |
| `commandId` | `String` | A single command — or: |
| `capabilityPackageId` / `capabilityId` | `String` | The Application and Skill to run (`application_id` / `skill_id` on the wire) |
| `executionParametersJson` | `String` | The run's input, as JSON |
| `autoStart` | `Boolean` | Whether the run starts when created |
| `organizationId` | `String` | The organization the schedule belongs to |

See [Scheduled triggers](../concepts/applications-and-skills.md#scheduled-triggers).

## Skill Registry

| Method | Returns | Purpose |
| --- | --- | --- |
| `observeSkillContract(SkillContractProtoDTO)` | `SkillContractProtoDTO` | Upsert a contract — new for a never-seen `(command_id, schema_version)` pair, or refreshes content/last-seen for one already known |
| `listSkillContracts(status, commandId)` | `List<SkillContractProtoDTO>` | List the registry, optionally filtered by `SkillContractStatus`. When `commandId` is set, returns that one command's full version history instead — `status` is then ignored, matching the RPC's own semantics. Either argument may be `null` |
| `setSkillContractStatus(id, SkillContractStatus)` | `SkillContractProtoDTO` | Move a contract through its lifecycle: `ACTIVE` / `DRAFT` / `DEPRECATED` / `RETIRED` |
| `setSkillContractPermissions(id, List<String>)` | `SkillContractProtoDTO` | Full replacement, not a merge, of the contract's `requiredPermissions` |

`setSkillContractPermissions` is **declarative only right now**: the platform stores a contract's
permissions and carries them over to new schema versions, but nothing enforces them yet.
Permission strings are free-form by design (e.g. `"mission.launch"`, `"role:pilot"`) rather than
drawn from a fixed vocabulary.

`SkillContractProtoDTO` also carries a `compatibility` verdict
(`NEW`/`COMPATIBLE`/`BREAKING`/`UNKNOWN`) computed server-side whenever an observation introduces a
new `schema_version` for an already-known `command_id`, comparing its input/output schema against
the previous version — `BREAKING` means a required input was added/changed, or an existing
input/output property was removed or retyped, so an existing authored Skill graph referencing the
previous version may now be invalid. `compatibilityNotes` carries the human-readable reasons (e.g.
`"required field 'zoom' added"`).

In practice, most adapters won't call these directly — the platform records the live `Capability`
snapshot an adapter returns from `EdgeAdapterService#getCapabilities` into the registry itself.
This API is for an adapter (or the Integration Hub, which mirrors its configured sinks in) that
wants to publish a contract proactively. See
[Edge Adapter Reference — Capability Reporting](edge-sdk-adapter-reference.md#capability-reporting)
for the live-snapshot side of this.

## Capabilities

Capability reporting isn't on `ConnectorService` — it's `getCapabilities(String sn)` on
`EdgeAdapterService`, which you implement yourself. See
[Edge Adapter Reference — Capability Reporting](edge-sdk-adapter-reference.md#capability-reporting).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction
- [Python reference](edge-sdk-python-connector-reference.md)
- [1.3 Connector reference](edge-sdk-connector-reference-1.3.md) — the end-of-life 1.3 interface
