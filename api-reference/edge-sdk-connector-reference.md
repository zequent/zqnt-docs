# Edge SDK — Connector API Reference

> **Beta preview:** an unmerged 2.0.x branch adds Skill Registry self-reporting to this interface
> and removes every Mission/Task method outright — see the
> [2.0.x Beta reference](edge-sdk-connector-reference-2.0.md) if you want to see where this is
> headed. Not on `main`/the current 1.3.x release yet.

Exhaustive method reference for `ConnectorService`. For a narrative introduction and worked
examples, see the [Connector guide](../edge-sdk/edge-sdk-connector.md).

All methods return a `CompletableFuture`. `ConnectorServiceImpl` logs the error and resolves to
`null` (entity methods) or `false` (delete methods) on a `hasErrors` response, rather than throwing
— see [Error Handling](../edge-sdk/edge-sdk-connector.md#error-handling).

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `getAssetBySn(sn)` | `AssetDTO` | Get asset by serial number |
| `getAssetById(id)` | `AssetDTO` | Get asset by ID |
| `getSubAssetBySn(sn)` | `SubAssetDTO` | Get sub-asset by serial number |
| `registerAsset(AssetDTO)` | `AssetDTO` | Register a new asset |
| `updateAsset(id, AssetDTO)` | `AssetDTO` | Update an existing asset |
| `deRegisterAsset(id)` | `Boolean` | Deregister an asset |

## Asset payloads

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsertAssetPayload(assetSn, subAssetSn, AssetPayloadDTO)` | `AssetPayloadDTO` | Create or update an asset payload |

## Tasks

| Method | Returns | Purpose |
| --- | --- | --- |
| `getTaskById(id)` | `TaskDTO` | Resolve a task the platform handed the adapter (via `prepareTask`/`startTask`) |
| `getTaskByFlightId(flightId)` | `TaskDTO` | Look up a task by the external flight ID the adapter itself assigned it |
| `updateTask(id, TaskDTO)` | `TaskDTO` | Persist adapter-computed fields (e.g. an uploaded flight-plan's `fileUrl`/`fileMd5`) or a status transition back onto the task the adapter is executing |
| `createTask(TaskDTO)` | `TaskDTO` | Create a task record |
| `deleteTask(id)` | `Boolean` | Delete a task record |

**In practice, only the read methods and `updateTask` are used by a real adapter** — confirmed
end-to-end against `edge-dji`, the one production Java adapter: it calls `getTaskById` to resolve
the task it was handed, then `updateTask` twice per run (once to persist `fileUrl`/`fileMd5` after
uploading the flight plan — the gRPC response doesn't return those fields, so this is the only way
to keep them — and again to set `status = TASK_RUNNING` before triggering the flight). There are
zero calls to `createTask`/`deleteTask` anywhere in that adapter. `createTask`/`deleteTask` exist on
the interface, but creating and deleting task records is a **Client SDK** (customer application)
responsibility in every real usage found.

## Missions

| Method | Returns | Purpose |
| --- | --- | --- |
| `getMissionById(id)` | `MissionDTO` | Get a mission by ID |
| `createMission(MissionDTO)` | `MissionDTO` | Create a mission |
| `updateMission(id, MissionDTO)` | `MissionDTO` | Update a mission |
| `deleteMission(id)` | `Boolean` | Delete a mission |

Unlike tasks, **no confirmed real-adapter usage of any Mission method** was found — mission
creation/management is a Client SDK (customer application) concern.

## Schedulers

| Method | Returns | Purpose |
| --- | --- | --- |
| `getSchedulerById(id)` | `SchedulerDTO` | Get scheduler by ID |
| `createScheduler(SchedulerDTO)` | `SchedulerDTO` | Create a new scheduler |
| `updateScheduler(id, SchedulerDTO)` | `SchedulerDTO` | Update an existing scheduler |
| `deleteScheduler(id)` | `Boolean` | Delete a scheduler |

## Organization

| Method | Returns | Purpose |
| --- | --- | --- |
| `getOrganizationById(id)` | `OrganizationDTO` | Get organization by ID |

## Capabilities

Capability reporting isn't on `ConnectorService` — it's `getCapabilities(String sn)` on
`EdgeAdapterService`, which you implement yourself. See
[Edge Adapter Reference — Capability Reporting](edge-sdk-adapter-reference.md#capability-reporting).
