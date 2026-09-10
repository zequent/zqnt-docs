# Edge SDK -- Mission Autonomy Service

`MissionAutonomyService` is a small, focused interface: it lets an edge adapter look up a **scheduler** definition directly from the Mission Autonomy service. Everything else related to running automated behavior on an asset — receiving task lifecycle calls, receiving commands, reporting progress — happens through other parts of the SDK, described below.

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

`cancelExecution(sn, externalExecutionId)` is the cancellation path for an asynchronously-running command you accepted via `sendCustomCommand`.

`MissionAutonomyService` exists for the one case where an adapter needs scheduler metadata directly:

```java
public interface MissionAutonomyService {
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

Scheduler CRUD (create/update/delete) is available through `ConnectorService` instead — see [Connector](edge-sdk-connector.md#schedulers).

---

## Where task execution actually happens

| Concern | Where it lives |
| --- | --- |
| Receiving `prepareTask`/`startTask`/`pauseTask`/`resumeTask`/`stopTask`/`cancelExecution` calls | `EdgeAdapterService` — see [Edge Adapter](edge-sdk-adapter.md#task-execution) |
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
