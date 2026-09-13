# Zequent Client SDK — Connector API Reference

Exhaustive method reference for `client.connector()`. For a narrative introduction, request-context
handling, and worked examples, see the [Connector guide](../client-sdk/CONNECTOR.md). For Python, see
[CONNECTOR_PYTHON.md](client-sdk-connector-python.md).

All methods return a `CompletableFuture`. `MissionResponse`/`TaskResponse`/`WaypointsResponse`/
`SchedulerResponse` use `isSuccess()` + `getError()`; everything else uses `getHasErrors()` +
`getError()` — see [Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md) for what these
actually mean.

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `getAssetBySn(GetAssetBySnRequest)` | `ConnectorResponse` | Look up an asset by its serial number |
| `getAssetById(GetAssetByIdRequest)` | `ConnectorResponse` | Look up an asset by its platform ID |
| `getSubAssetBySn(GetSubAssetBySnRequest)` | `ConnectorResponse` | Look up a sub-asset (e.g. the drone paired to a dock) |
| `registerAsset(RegisterAssetRequest)` | `ConnectorResponse` | Register a new asset (normally done by an edge adapter, not a customer app) |
| `updateAsset(UpdateAssetRequest)` | `ConnectorResponse` | Update asset metadata |
| `updateSubAsset(UpdateSubAssetRequest)` | `ConnectorResponse` | Update sub-asset metadata |
| `deregisterAsset(DeregisterAssetRequest)` | `ConnectorResponse` | Remove an asset |

## Asset payloads

Arbitrary, versioned metadata blobs attached to an asset (e.g. flight-plan artifacts, calibration data).

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsertAssetPayload(UpsertAssetPayloadRequest)` | `AssetPayloadResponse` | Create or update a payload |
| `listAssetPayloads(ListAssetPayloadsRequest)` | `AssetPayloadListResponse` | List payloads for an asset |
| `deleteAssetPayload(DeleteAssetPayloadRequest)` | `AssetPayloadResponse` | Remove a payload |

## Organizations

| Method | Returns | Purpose |
| --- | --- | --- |
| `getOrganization(GetOrganizationRequest)` | `ConnectorResponse` | Get the calling application's organization info |

## Missions

Mission methods create and read mission **records** — creating one does not fly anything by itself.
How a flight is actually triggered depends on the adapter; see
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use).

> **Prefer [`missionAutonomy()`'s copies](client-sdk-mission-autonomy.md#missions) of `createMission`/`updateMission` for anything real.**
> Confirmed against the backend: they run the mission through route optimization before writing the
> exact same record these methods write directly, unoptimized. Use these only if you specifically
> want the raw write.

| Method | Returns | Purpose |
| --- | --- | --- |
| `getMission(GetMissionRequest)` | `MissionResponse` | Get a mission by ID |
| `createMission(CreateMissionRequest)` | `MissionResponse` | Create a mission |
| `updateMission(UpdateMissionRequest)` | `MissionResponse` | Update a mission |
| `deleteMission(DeleteMissionRequest)` | `MissionResponse` | Delete a mission |
| `uploadMissionNfzZones(UploadMissionZonesRequest)` | `MissionResponse` | Attach no-fly zones to a mission |

### `GeoAreaDTO.type` requirements

| Type | Requires |
| --- | --- |
| `GEO_AREA_TYPE_POLYGON` | 3+ `vertices` |
| `GEO_AREA_TYPE_BOUNDING_BOX` | exactly 2 `vertices` |
| `GEO_AREA_TYPE_CIRCLE` | `center` + a positive `radiusMeters` |
| `GEO_AREA_TYPE_GEO_JSON` | `geoJson` |

`UploadMissionZonesRequest.replaceExisting(true)` replaces the mission's whole zone set instead of
appending to it.

## Tasks

Task methods create and read task **records** — creating a task does not start it. `startTask`
(on `MissionAutonomy`, not `Connector`) reaches the device only where the adapter implements the
task methods (DJI, SAPIENT) — see
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use).

> **Prefer [`missionAutonomy()`'s copies](client-sdk-mission-autonomy.md#tasks) of `createTask`/`updateTask` for a waypoint task.**
> Confirmed against the backend: for a task with a `missionId` and waypoints, they route the task
> around that mission's no-fly zones before writing the exact same record these methods write
> directly, unoptimized.

| Method | Returns | Purpose |
| --- | --- | --- |
| `getTask(GetTaskRequest)` | `TaskResponse` | Get a task by ID |
| `getTaskByFlightId(GetTaskByFlightIdRequest)` | `TaskResponse` | Get a task by its external flight ID (`TaskDTO.externalTaskId` — the ID an edge adapter assigned it) |
| `getWaypointsByTaskId(GetWaypointsByTaskIdRequest)` | `WaypointsResponse` | Get the resolved waypoint list for a task |
| `createTask(CreateTaskRequest)` | `TaskResponse` | Create a task |
| `updateTask(UpdateTaskRequest)` | `TaskResponse` | Update a task |
| `deleteTask(DeleteTaskRequest)` | `TaskResponse` | Delete a task |

## Schedulers

Schedulers define when and how often a task or command runs.

| Method | Returns | Purpose |
| --- | --- | --- |
| `getScheduler(GetSchedulerRequest)` | `SchedulerResponse` | Get a scheduler by ID |
| `createScheduler(CreateSchedulerRequest)` | `SchedulerResponse` | Create a scheduler |
| `createSchedulers(CreateSchedulersRequest)` | `SchedulerResponse` | Create several schedulers in one call |
| `updateScheduler(UpdateSchedulerRequest)` | `SchedulerResponse` | Update a scheduler |
| `deleteScheduler(DeleteSchedulerRequest)` | `SchedulerResponse` | Delete a scheduler |
| `deleteSchedulers(DeleteSchedulersRequest)` | `SchedulerResponse` | Delete several schedulers in one call |
| `deleteSchedulersByTask(DeleteSchedulersByTaskRequest)` | `SchedulerResponse` | Delete every scheduler tied to one task |

## Technical configuration & policies

Read-only lookups useful when your application needs to mirror platform-side configuration or
operational policy (e.g. no-fly zones, altitude limits) rather than hard-coding it.

| Method | Returns | Purpose |
| --- | --- | --- |
| `getTechnicalConfigs(GetTechnicalConfigsRequest)` | `ConnectorConfigResponse` | Fetch technical configuration values |
| `getActivePoliciesByType(GetPoliciesRequest)` | `ConnectorPolicyResponse` | Fetch active operational policies of a given type |
| `getAllActivePolicies(GetAllActivePoliciesRequest)` | `ConnectorPolicyResponse` | Fetch every active operational policy |

## Asset monitoring and batch ingestion

Four methods with no javadoc and **no confirmed real caller found** anywhere in this SDK's own
test suite or in `edge-dji` — documented here from their verified type signatures and
implementation, not from any stated intent. Take the "purpose" column as inferred from behavior,
not as a confirmed use case.

| Method | Returns | Behavior |
| --- | --- | --- |
| `assetMonitoring(AssetMonitoringRequest, onData, onError)` | `StreamHandle` | Server-streamed subscription: pushes a full `List<AssetDTO>` snapshot on `onData` every time the platform's asset set changes, using this SDK's own `com.zqnt.sdk.client.livedata.domains.StreamHandle` — the same reconnecting-handle class the SDK's Live Data telemetry streaming returns — call `.stop()`/`.close()` to cancel. No Java prose doc exists for that handle's start/stop/reconnect semantics yet; see the [Python asyncio streaming pattern](../client-sdk/ASYNCIO.md#streaming) for the same paradigm (Python's own `StreamHandle`, not literally this class) worked through from the client side. Conceptually the org-scoped counterpart of the Python Edge SDK's `ConnectorClient.watch_assets()`, which does the equivalent at the edge. |
| `storeTelemetryBatch()` | `ConnectorBatchSession<StoreTelemetryRequest>` | Opens a client-streaming batch session: call `send(...)` repeatedly, then `complete()` for one aggregated `ConnectorResponse`. Each `StoreTelemetryRequest` carries a `sourceType` (`ASSET`/`SUB_ASSET`), `assetId`, `sourceSystem`, and a `TelemetryData` payload — this writes telemetry *as* if from an edge adapter, from client-side code instead |
| `storeDetectionBatch()` | `ConnectorBatchSession<StoreDetectionRequest>` | Same batch-session pattern; each request carries one `DetectionDTO` |
| `storeNotificationBatch()` | `ConnectorBatchSession<StoreNotificationRequest>` | Same batch-session pattern; each request carries one of an `AssetStatusEvent`, `TaskEvent`, or `MissionEvent` (discriminated by `eventType`), plus a `Severity` |

`ConnectorBatchSession<T>` (`send`/`complete`/`cancel`/`isCompleted`, `AutoCloseable`) is the same
client-streaming session shape across all three batch methods — open one, `send()` as many records
as you have, then `complete()` once to get back a single `ConnectorResponse` for the whole batch.

Read these as a bulk/backfill ingestion path for telemetry, detections, and notifications that
didn't originate from a normal edge-adapter Live Data stream — e.g. importing historical data or
bridging a third-party data source — rather than something a typical real-time integration needs.
If you're building a normal edge adapter, use
[Live Data](../edge-sdk/edge-sdk-live-data.md) instead; these three exist on the **Client** SDK, not
the Edge SDK.

## Capabilities

Capability discovery lives on `client.remoteControl()`, not `client.connector()` — see
[Remote Control — Capabilities & custom commands](../client-sdk/REMOTE_CONTROL.md#capabilities--custom-commands).
What an asset reports comes from its edge adapter — see
[Edge SDK — Connector](../edge-sdk/edge-sdk-connector.md#capabilities).
