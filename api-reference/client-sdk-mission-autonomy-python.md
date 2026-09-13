# Zequent Client SDK (Python) — Mission Autonomy API Reference

> **Beta preview:** an unmerged 2.0.x branch replaces this entire interface with an
> Application → Skill → SkillExecution model — see the
> [2.0.x Beta reference](client-sdk-mission-autonomy-python-2.0.md) if you want to see where this is
> headed. Not on `main`/the current 1.3.x release yet.

Exhaustive method reference for `client.mission_autonomy`. For a narrative introduction, see the
[Mission Autonomy guide](../client-sdk/MISSION_AUTONOMY_PYTHON.md). For Java, see
[client-sdk-mission-autonomy.md](client-sdk-mission-autonomy.md); for Go,
[client-sdk-mission-autonomy-go.md](client-sdk-mission-autonomy-go.md).

Every method is a coroutine. `MissionResponse`/`TaskResponse`/`SchedulerResponse` carry
`success: bool` and `error: ErrorInfo | None` — they don't raise for a business-level error, only
for a transport failure (`grpc.aio.AioRpcError`). See
[Mission Autonomy — Error handling](../client-sdk/MISSION_AUTONOMY_PYTHON.md#error-handling).

`client.connector` has **no** Mission/Task methods at all in this SDK — unlike Java (where both
interfaces have overlapping copies), `client.mission_autonomy` is the only way to manage missions
and tasks in Python, the same as the Go SDK.

## Missions

| Method | Returns | Notes |
| --- | --- | --- |
| `create_mission(mission: MissionDTO)` | `MissionResponse` | **Route-optimized** — confirmed against the backend `MissionAutonomyGrpcService`: runs the mission through `missionRouteOptimizer.optimize(...)` before writing it |
| `update_mission(mission_id, mission)` | `MissionResponse` | Same optimization as `create_mission` |
| `get_mission(mission_id)` | `MissionResponse` | Plain passthrough — no optimization applies to reads |
| `delete_mission(mission_id)` | `MissionResponse` | |

## Tasks

| Method | Returns | Notes |
| --- | --- | --- |
| `create_task(task: TaskDTO)` | `TaskResponse` | Route-optimized for a waypoint task with a `mission_id` (no-fly-zone-aware expansion), same as `create_mission` |
| `update_task(task_id, task)` | `TaskResponse` | Same optimization as `create_task` |
| `get_task(task_id)` | `TaskResponse` | Plain passthrough |
| `get_task_by_flight_id(flight_id)` | `TaskResponse` | Look up by the external flight ID an edge adapter assigned the task |
| `delete_task(task_id)` | `TaskResponse` | |

### Task execution lifecycle

| Method | Returns |
| --- | --- |
| `start_task(task_id)` | `TaskResponse` |
| `stop_task(task_id)` | `TaskResponse` |
| `pause_task(task_id)` | `TaskResponse` |
| `resume_task(task_id)` | `TaskResponse` |

These forward a bare task ID to the adapter — works only where the adapter implements the task
lifecycle. See [Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use)
for the per-adapter picture, including the command-based alternative that doesn't use this lifecycle
at all.

## Schedulers

| Method | Returns | Notes |
| --- | --- | --- |
| `create_scheduler(scheduler: SchedulerDTO)` | `SchedulerResponse` | |
| `create_schedulers(schedulers: list[SchedulerDTO])` | `SchedulerResponse` | Create several in one call |
| `get_scheduler(scheduler_id)` | `SchedulerResponse` | |
| `update_scheduler(scheduler_id, scheduler)` | `SchedulerResponse` | |
| `delete_scheduler(scheduler_id)` | `SchedulerResponse` | |
| `delete_schedulers(scheduler_ids: list[str])` | `SchedulerResponse` | Delete several in one call |
| `list_schedulers(task_id: str \| None = None)` | `SchedulerResponse` | Result is in `.schedulers`. `task_id` omitted/`None` lists every scheduler, unfiltered |

Identical scheduler methods (same wire messages, same non-raising response convention) also exist on
`client.connector` — see [Connector — Error handling](../client-sdk/CONNECTOR_PYTHON.md#error-handling).
No optimization difference either way; scheduler operations aren't route-related.
