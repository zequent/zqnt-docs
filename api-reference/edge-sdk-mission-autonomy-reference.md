# Edge SDK — Mission Autonomy API Reference

> **Beta preview:** an unmerged 2.0.x branch shrinks this interface to `getScheduler` alone — see
> the [2.0.x Beta reference](edge-sdk-mission-autonomy-reference-2.0.md) if you want to see where
> this is headed. Not on `main`/the current 1.3.x release yet.

Exhaustive method reference for `MissionAutonomyService`. For a narrative introduction, see the
[Mission Autonomy guide](../edge-sdk/edge-sdk-mission-autonomy.md).

**No confirmed real-adapter usage of any method on this interface.** `edge-dji` (the one production
Java adapter) wires up a `MissionAutonomyService` bean via its own CDI producer, but calls none of
its six methods anywhere — its one Mission Autonomy-shaped call
(`getTaskByFlightId`) actually goes through `ConnectorService`, not this interface. Confirmed by
grepping for every method name across the adapter's source.

```java
public interface MissionAutonomyService {
    CompletableFuture<MissionDTO> createMission(CreateMissionRequest createMissionRequest);
    CompletableFuture<MissionDTO> updateMission(UpdateMissionRequest updateMissionRequest);
    CompletableFuture<MissionDTO> getMission(GetMissionRequest getRequest);
    CompletableFuture<TaskDTO> getTask(GetTaskRequest getTaskRequest);
    CompletableFuture<TaskDTO> getTaskByFlightId(GetTaskByFlightIdRequest getTaskRequest);
    CompletableFuture<SchedulerDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

## Missions

| Method | Returns | Purpose |
| --- | --- | --- |
| `createMission(CreateMissionRequest)` | `MissionDTO` | Create a mission — **route-optimized**, see below |
| `updateMission(UpdateMissionRequest)` | `MissionDTO` | Update a mission — same optimization |
| `getMission(GetMissionRequest)` | `MissionDTO` | Get a mission by ID (`request.getMissionId()`) |

`createMission`/`updateMission` are confirmed, via `MissionAutonomyGrpcService` on the platform side,
to run the mission through `missionRouteOptimizer.optimize(...)` before delegating to
`connector-service` — the exact same optimization the Client SDK's `client.missionAutonomy()`
applies. This is the one place this interface differs functionally from
[`ConnectorService`](edge-sdk-connector-reference.md)'s own `createMission`/`updateMission`, which write the
unoptimized record directly.

## Tasks

| Method | Returns | Purpose |
| --- | --- | --- |
| `getTask(GetTaskRequest)` | `TaskDTO` | Get a task by ID (`request.getTaskId()`) |
| `getTaskByFlightId(GetTaskByFlightIdRequest)` | `TaskDTO` | Get a task by its external flight ID (`request.getFlightId()`) |

Confirmed, on the platform side, to be a **plain passthrough** to `connector-service` — no
optimization applies to reads. Functionally identical to calling
[`ConnectorService.getTaskById`/`getTaskByFlightId`](edge-sdk-connector-reference.md#tasks) directly; this
interface adds nothing for these two beyond a different gRPC endpoint to reach the same result.

## Schedulers

| Method | Returns | Purpose |
| --- | --- | --- |
| `getScheduler(GetSchedulerRequest)` | `SchedulerDTO` | Get a scheduler by ID (`request.getSchedulerId()`) |

Also a plain passthrough to `connector-service` — identical in effect to
[`ConnectorService.getSchedulerById`](edge-sdk-connector-reference.md#schedulers). Scheduler create/update/delete
stay on `ConnectorService` only; this interface has no write methods for schedulers at all.
