# Edge SDK -- Mission Autonomy Service

`MissionAutonomyService` gives an edge adapter six methods — mission create/update/get, task get (by ID or flight ID), and scheduler get — reaching the platform's `mission-autonomy-service` directly instead of `connector-service`. **No confirmed real-adapter usage of any of them**: `edge-dji`, the one production Java adapter, wires up a `MissionAutonomyService` bean but calls none of its methods anywhere in its source. Everything related to actually running automated behavior on an asset — receiving task lifecycle calls, receiving commands, reporting progress — happens through other parts of the SDK, described below.

Full method-by-method reference, including which methods are route-optimized and which are plain
passthroughs: [Mission Autonomy API Reference](../api-reference/edge-sdk-mission-autonomy-reference.md).

## Table of Contents

- [Overview](#overview)
- [MissionAutonomyService Interface](#missionautonomyservice-interface)
- [Where task execution actually happens](#where-task-execution-actually-happens)
- [Configuration](#configuration)

---

## Overview

Most edge adapters never call `MissionAutonomyService` directly. The platform drives execution by calling *into* your adapter, and your adapter reports progress back over the Live Data connection — it doesn't poll or manage missions itself.

**There are two ways that call can arrive, and adapters differ in which they support.**

- **Task-based.** `prepareTask` / `startTask` / `pauseTask` / `resumeTask` / `stopTask` receive a bare task ID. Your adapter resolves it with `ConnectorService.getTaskById(...)` and reads the `WaypointTaskConfig` off the returned `TaskDTO`. This is what the DJI adapter does — `startTask` fetches the Task, builds the KMZ and executes it on the dock. SAPIENT also implements these, because its own protocol owns the task the ID refers to.
- **Command-based.** The platform calls `sendCustomCommand` with a `command_id` such as `mission.waypoint.execute` and the full parameter set inline, so no lookup is needed. This is what the MAVLink adapter and the simulator do.

Pick whichever suits your device and be explicit about it in your capability advertisement. Note that the Go Edge SDK's connector client has no task lookup, so a Go adapter can only take the command-based route. Anything you leave unimplemented returns `NOT_IMPLEMENTED`, which is a valid and common choice.

To stop a running task, the platform calls `stopTask` on your adapter.

### MissionAutonomyService Interface

`MissionAutonomyService` has six methods — mission create/update/get, task get (by ID or flight ID),
and scheduler get — reaching `mission-autonomy-service` directly rather than `connector-service`.
None have confirmed real-adapter usage; the interface exists for an adapter that wants a
route-optimized mission write, or a read that happens to already be wired through this service
rather than `ConnectorService`:

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

```java
import com.zqnt.utils.mission.proto.GetSchedulerRequest;

GetSchedulerRequest request = GetSchedulerRequest.newBuilder()
    .setSchedulerId("scheduler-uuid")
    .build();

missionAutonomyService.getScheduler(request)
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler))
    .exceptionally(err -> {
        log.error("Failed to get scheduler", err);
        return null;
    });
```

`createMission`/`updateMission` are route-optimized; every other method here (including this
`getScheduler` example) is a plain passthrough to `connector-service` — functionally identical to
calling the equivalent [`ConnectorService`](edge-sdk-connector.md) method directly. See the
[reference](../api-reference/edge-sdk-mission-autonomy-reference.md) for the full breakdown. Scheduler
create/update/delete is available only through `ConnectorService` — see
[Connector](edge-sdk-connector.md#schedulers).

> **Beta preview — 2.0.x, not yet released.** An unmerged branch shrinks this interface to
> `getScheduler` alone — every Mission/Task method above is gone outright, not deprecated. Given
> [no confirmed real-adapter usage](#overview) of any of them today, this is unlikely to affect a
> real adapter migrating forward. See the
> [2.0.x migration guide](../concepts/migration-guide-2.0.md#per-sdk-impact) or the
> [2.0.x reference](../api-reference/edge-sdk-mission-autonomy-reference-2.0.md) directly.

---

## Where task execution actually happens

| Concern | Where it lives |
| --- | --- |
| Receiving `prepareTask`/`startTask`/`pauseTask`/`resumeTask`/`stopTask` calls | `EdgeAdapterService` — see [Edge Adapter Reference](../api-reference/edge-sdk-adapter-reference.md#task-execution) |
| Reporting progress/telemetry while a task runs | `LiveDataService` — see [Live Data](edge-sdk-live-data.md) |
| Declaring which commands your adapter supports | `getCapabilities` on `EdgeAdapterService` — see [Connector](edge-sdk-connector.md#capabilities) |
| Creating missions and tasks, and triggering them | The **Client SDK**, used by customer applications |

A typical `prepareTask` implementation looks up whatever it needs (e.g. a stored flight plan) via `ConnectorService`'s asset payload methods, rather than through `MissionAutonomyService`:

```java
@Override
public CompletableFuture<CommandResult> prepareTask(String taskId, String tid) {
    // Fetch whatever your adapter needs to execute this task — e.g. a previously
    // uploaded flight plan stored as an asset payload — then stage it on the device.
    return CompletableFuture.completedFuture(
        CommandResult.success("Task prepared", tid, taskId)
    );
}
```

---

## Configuration

```properties
quarkus.grpc.clients.mission-autonomy-service.host=localhost
quarkus.grpc.clients.mission-autonomy-service.port=8004
quarkus.grpc.clients.mission-autonomy-service.keep-alive-without-calls=true
```

For container deployments:

```properties
quarkus.grpc.clients.mission-autonomy-service.host=mission-autonomy-service
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.
