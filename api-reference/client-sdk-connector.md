# Zequent Client SDK — Connector API Reference

Method reference for `client.connector()`. For a narrative introduction and worked examples, see the
[Connector guide](../client-sdk/CONNECTOR.md). For Python, see
[client-sdk-connector-python.md](client-sdk-connector-python.md); for Go,
[client-sdk-connector-go.md](client-sdk-connector-go.md).

All request/response methods return a `CompletableFuture`. Response objects carry `isSuccess()` and
`getError()` (`errorCode`, `errorMessage`) for business errors such as "not found" — see
[Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md). The Skill Registry methods work on raw
generated proto types and complete exceptionally with a `ConnectorClientException`
(`getErrorCode()`, `getTransactionId()`) instead.

## What a client credential may call

A customer application authenticates with a client credential (`ZQNT_CLIENT_TOKEN`). The platform
binds it to one organization and allows only part of this interface. Everything else is
administration (done in the Admin Console) or belongs to edge adapters and the platform's own
services.

- A method marked **refused** completes exceptionally with a gRPC `PERMISSION_DENIED`.
- A method that names an asset (by serial number or ID) which is not your organization's is refused
  with `PERMISSION_DENIED` too. An asset that does not exist gets the same answer.
- Refusals are not retried by the SDK.

## Assets

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `getAssetBySn(GetAssetBySnRequest)` | `ConnectorResponse` | allowed | Look up an asset by serial number (`context.sn`) |
| `getAssetById(GetAssetByIdRequest)` | `ConnectorResponse` | allowed | Look up an asset by platform ID |
| `getSubAssetBySn(GetSubAssetBySnRequest)` | `ConnectorResponse` | allowed | Look up a sub-asset (e.g. the drone paired to a dock) by serial number (`context.sn`) |
| `updateAsset(UpdateAssetRequest)` | `ConnectorResponse` | allowed | Update asset metadata; `updateFields` limits which fields are written |
| `updateSubAsset(UpdateSubAssetRequest)` | `ConnectorResponse` | allowed | Update sub-asset metadata |
| `deregisterAsset(DeregisterAssetRequest)` | `ConnectorResponse` | allowed | Remove an asset, addressed by serial number (`context.sn`) |
| `registerAsset(RegisterAssetRequest)` | `ConnectorResponse` | refused | Assets are added in the Admin Console or paired by their edge adapter with a pairing code |

`ConnectorResponse` carries the result in `asset` or `subAsset`, plus `success`, `tid` and `error`.

## Asset payloads

Hardware mounted on an asset or sub-asset (a camera, gimbal or sensor): slot, name, serial number,
kind, vendor, model, firmware version and state. A Java edge adapter can also report them through the Edge SDK. The owner is a
`PayloadOwner` (`type` `ASSET` or `SUB_ASSET`, plus its `id`).

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `upsertAssetPayload(UpsertAssetPayloadRequest)` | `AssetPayloadResponse` | allowed | Create or update a payload; `updateFields` limits which fields are written |
| `listAssetPayloads(ListAssetPayloadsRequest)` | `AssetPayloadListResponse` | allowed | List an owner's payloads (`owner`, or `context.assetId`/`context.sn`) |

## Skill Registry

The platform-wide catalog of every command an edge adapter has reported, one entry per
`(commandId, schemaVersion)`. Types are the generated `com.zqnt.utils.connector.proto.SkillContractProtoDTO`
and `SkillContractStatus` (`ACTIVE`, `DRAFT`, `DEPRECATED`, `RETIRED`).

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `listSkillContracts(SkillContractStatus status, String commandId)` | `List<SkillContractProtoDTO>` | allowed | List the registry. `status` filters by lifecycle state; `commandId` returns that command's full version history instead (status is then ignored). Either may be `null` |
| `observeSkillContract(SkillContractProtoDTO)` | `SkillContractProtoDTO` | refused | Record a contract (done by the platform when adapters report capabilities) |
| `setSkillContractStatus(String id, SkillContractStatus)` | `SkillContractProtoDTO` | refused | Change a contract's lifecycle state |
| `setSkillContractPermissions(String id, List<String>)` | `SkillContractProtoDTO` | refused | Replace a contract's required permissions |

## Administration (refused for a client credential)

These are on the interface, but a client credential gets `PERMISSION_DENIED`. Manage them in the
Admin Console.

| Method | Returns | Purpose |
| --- | --- | --- |
| `getOrganization(GetOrganizationRequest)` | `ConnectorResponse` | Organization info |
| `getScheduler` / `createScheduler` / `createSchedulers` / `updateScheduler` / `deleteScheduler` / `deleteSchedulers` | `SchedulerResponse` | Schedules that run an Application — see [Applications & Skills](../concepts/applications-and-skills.md) |
| `getActivePoliciesByType(GetPoliciesRequest)` / `getAllActivePolicies(GetAllActivePoliciesRequest)` | `ConnectorPolicyResponse` | Operational policies (asset selection) |
| `getTechnicalConfigs(GetTechnicalConfigsRequest)` | `ConnectorConfigResponse` | Technical configuration values — see [Configuration](../concepts/configuration.md) |

## Batch ingestion (refused for a client credential)

`storeTelemetryBatch()`, `storeDetectionBatch()` and `storeNotificationBatch()` open a
client-streaming `ConnectorBatchSession<T>` (`send`, `complete`, `cancel`, `isCompleted`,
`AutoCloseable`). They are the platform's own persistence path for data coming from edge adapters,
and a client credential is refused. Edge adapters report telemetry, detections and command execution
events through the Edge SDK — see [Live Data](../edge-sdk/edge-sdk-live-data.md).

## Capabilities

What a specific asset can do right now is read from `client.remoteControl().getCapabilities(sn)` —
see [Remote Control — Capabilities & custom commands](../client-sdk/REMOTE_CONTROL.md#capabilities--custom-commands).
What an asset reports comes from its edge adapter — see
[Edge SDK — Connector](../edge-sdk/edge-sdk-connector.md#capabilities).
