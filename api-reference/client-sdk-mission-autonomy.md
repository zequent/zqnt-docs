# Zequent Client SDK — Mission Autonomy API Reference

> **Beta preview:** an unmerged 2.0.x branch replaces this entire interface with an
> Application → Skill → SkillExecution model — see the
> [2.0.x Beta reference](client-sdk-mission-autonomy-2.0.md) if you want to see where this is
> headed. Not on `main`/the current 1.3.x release yet.

Exhaustive method reference for `client.missionAutonomy()`. All methods return a `CompletableFuture`;
`MissionResponse`/`TaskResponse`/`SchedulerResponse` use `isSuccess()` + `getError()` — see
[Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md).

## This is not just a second copy of Connector's CRUD

`client.connector()` also has `createMission`/`createTask`/`updateMission`/`updateTask` and the rest
of Mission/Task/Scheduler CRUD (see the
[Connector reference](client-sdk-connector.md#missions)) — the two look redundant, but they aren't
equivalent. Confirmed against the real backend
(`MissionAutonomyGrpcService`, `core/services/mission-autonomy`):

- `missionAutonomy().createMission`/`updateMission` run the mission through
  `missionRouteOptimizer.optimize(...)` **before** delegating to the exact same connector-service
  call `connector()` would make directly.
- `missionAutonomy().createTask`/`updateTask` (for a waypoint task with a `missionId`) run
  `expandAndOptimize(...)` — routing the task's waypoints around that mission's no-fly zones —
  before the same delegation.
- `connector()`'s versions skip both of these. They write the same record, but with no
  optimization or NFZ-aware expansion applied.

**Use `missionAutonomy()` for missions and waypoint tasks in the normal case.** Reach for
`connector()`'s copies only if you specifically want the raw, unoptimized write.

## Missions

| Method | Returns | Purpose |
| --- | --- | --- |
| `createMission(MissionDTO)` | `MissionResponse` | Create a mission (route-optimized) |
| `updateMission(missionId, MissionDTO)` | `MissionResponse` | Update a mission (route-optimized) |
| `getMission(missionId)` | `MissionResponse` | Get a mission by ID |
| `deleteMission(missionId)` | `MissionResponse` | Delete a mission |
| `uploadMissionNfzZones(missionId, List<MissionZoneDTO>, replaceExisting)` | `MissionResponse` | Attach no-fly zones to a mission — do this before creating tasks against it, so task creation/update can actually route around them |

## Tasks

| Method | Returns | Purpose |
| --- | --- | --- |
| `createTask(TaskDTO)` | `TaskResponse` | Create a task (NFZ-expanded/optimized if it's a waypoint task with a `missionId`) |
| `updateTask(taskId, TaskDTO)` | `TaskResponse` | Update a task (same optimization applied) |
| `getTask(taskId)` | `TaskResponse` | Get a task by ID |
| `getTaskByFlightId(flightId)` | `TaskResponse` | Get a task by its external flight ID |
| `deleteTask(taskId)` | `TaskResponse` | Delete a task |

### Task execution lifecycle — exclusive to this interface

| Method | Returns | Purpose |
| --- | --- | --- |
| `startTask(taskId)` | `TaskResponse` | Start executing a task |
| `stopTask(taskId)` | `TaskResponse` | Stop a running task |
| `pauseTask(taskId)` | `TaskResponse` | Pause a running task |
| `resumeTask(taskId)` | `TaskResponse` | Resume a paused task |

`Connector` has no equivalent of any of these four — this is the only interface that can actually
trigger, halt, or resume a task on the device. They forward a bare task ID to the adapter, which
works only where the adapter implements the task lifecycle (DJI, SAPIENT) — see
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use) for the
full per-adapter picture, including the command-based alternative (MAVLink, the simulator) that
doesn't use this lifecycle at all.

## Schedulers

| Method | Returns | Purpose |
| --- | --- | --- |
| `createScheduler(SchedulerDTO)` | `SchedulerResponse` | Create a scheduler |
| `updateScheduler(schedulerId, SchedulerDTO)` | `SchedulerResponse` | Update a scheduler |
| `getScheduler(schedulerId)` | `SchedulerResponse` | Get a scheduler by ID |
| `deleteScheduler(schedulerId)` | `SchedulerResponse` | Delete a scheduler |
| `createSchedulers(List<SchedulerDTO>)` | `SchedulerResponse` | Create several schedulers in one call |
| `deleteSchedulers(List<String>)` | `SchedulerResponse` | Delete several schedulers in one call |
| `deleteAllSchedulersByTaskId(taskId)` | `SchedulerResponse` | Delete every scheduler tied to one task |

Unlike Missions/Tasks, no optimization pass applies here — scheduler CRUD is identical whether
called through `missionAutonomy()` or `connector()` (the latter's scheduler methods aren't
route-related, so there's nothing to optimize either way).
