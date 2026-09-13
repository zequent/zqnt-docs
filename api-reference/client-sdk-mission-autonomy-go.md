# Zequent Client SDK (Go) — Mission Autonomy API Reference

Exhaustive method reference for `missionautonomy.New(conn)`. For a narrative introduction, see the
[Quickstart](../client-sdk/QUICKSTART_GO.md#missionautonomy--missions-tasks--schedulers). For Java,
see [client-sdk-mission-autonomy.md](client-sdk-mission-autonomy.md).

Every method takes a `context.Context` first; most return `(*Result, error)` with no separate
`HasErrors` flag — a non-nil `error` already carries the platform-side message.

Unlike the Java/Python SDKs, **Go's `connector.New(conn)` has no Mission/Task methods at all** — see
[Connector Reference](client-sdk-connector-go.md). There is no "which one should I call" question
here: `missionautonomy` is the only place to create, read, update, or delete missions and tasks, and
the only place to drive a task's lifecycle.

## Missions

| Method | Returns | Notes |
| --- | --- | --- |
| `CreateMission(ctx, mission *missionautonomydto.MissionProtoDTO)` | `*MissionProtoDTO` | **Route-optimized** — confirmed against the backend `MissionAutonomyGrpcService`: runs the mission through `missionRouteOptimizer.optimize(...)` before writing it |
| `UpdateMission(ctx, missionID, mission)` | `*MissionProtoDTO` | Same optimization as `CreateMission` |
| `GetMission(ctx, missionID)` | `*MissionProtoDTO` | Plain passthrough — no optimization applies to reads |
| `DeleteMission(ctx, missionID) error` | — | |

## Tasks

| Method | Returns | Notes |
| --- | --- | --- |
| `CreateTask(ctx, task *missionautonomydto.TaskProtoDTO)` | `*TaskProtoDTO` | Route-optimized for a waypoint task with a `MissionId` (no-fly-zone-aware expansion), same as `CreateMission` |
| `UpdateTask(ctx, taskID, task)` | `*TaskProtoDTO` | Same optimization as `CreateTask` |
| `GetTask(ctx, taskID)` | `*TaskProtoDTO` | Plain passthrough |
| `GetTaskByFlightID(ctx, flightID)` | `*TaskProtoDTO` | Look up by the external flight ID (e.g. a DJI/MAVLink mission identifier) an edge adapter assigned the task |
| `DeleteTask(ctx, taskID) error` | — | |

### Task execution lifecycle

| Method | Returns |
| --- | --- |
| `StartTask(ctx, taskID)` | `*TaskProtoDTO` |
| `StopTask(ctx, taskID)` | `*TaskProtoDTO` |
| `PauseTask(ctx, taskID)` | `*TaskProtoDTO` |
| `ResumeTask(ctx, taskID)` | `*TaskProtoDTO` |

These forward a bare task ID to the adapter — works only where the adapter implements the task
lifecycle. See [Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use)
for the per-adapter picture, including the command-based alternative that doesn't use this lifecycle
at all.

## Schedulers

| Method | Returns | Notes |
| --- | --- | --- |
| `ListSchedulers(ctx, taskID string)` | `[]*SchedulerProtoDTO` | `taskID == ""` lists every scheduler, unfiltered; a non-empty `taskID` filters to that task's schedulers. Lives here rather than on `connector.Client` because `ConnectorService` has no `ListSchedulers` RPC at this contract version |

All other scheduler CRUD (`Get`/`Create`/`Update`/`Delete`) lives on
[`connector.New(conn)`](client-sdk-connector-go.md#schedulers) instead — identical wire messages,
no optimization difference, since scheduler operations aren't route-related.
