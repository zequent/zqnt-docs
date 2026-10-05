# Edge SDK -- Mission Autonomy Service

`MissionAutonomyService` gives an edge adapter one method: `getScheduler`, which reads a scheduler's
definition from the platform's `mission-autonomy-service`. Everything related to actually running
automated behavior on an asset — receiving commands, reporting their progress — happens through other
parts of the SDK, described below.

Full method-by-method reference: [Mission Autonomy API Reference](../api-reference/edge-sdk-mission-autonomy-reference.md).

## Table of Contents

- [Overview](#overview)
- [MissionAutonomyService Interface](#missionautonomyservice-interface)
- [How Skill executions reach your adapter](#how-skill-executions-reach-your-adapter)
- [Configuration](#configuration)

---

## Overview

Most edge adapters never call `MissionAutonomyService` directly. Automated work is authored as
Applications and Skills, and the platform runs a Skill by calling *into* your adapter, one command at a
time; your adapter reports each command's progress back over the Live Data connection. It doesn't poll
or manage executions itself. See [Applications & Skills](../concepts/applications-and-skills.md).

The 1.3 Mission and Task methods are gone from this interface, and the platform no longer starts work
through an adapter's task methods — see
[Upgrading from 1.3](../concepts/migration-guide.md#task-based-execution-is-gone).

### MissionAutonomyService Interface

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

A scheduler now targets a Skill or a single command directly — see the
[reference](../api-reference/edge-sdk-mission-autonomy-reference.md). `ConnectorService.getSchedulerById`
reads a scheduler too, and scheduler create/update/delete is available only through `ConnectorService` —
see [Connector](edge-sdk-connector.md#schedulers).

Like the other SDK services, your adapter creates it —
`new MissionAutonomyServiceImpl(MissionAutonomyServiceGrpc.newStub(channel), protoJsonMapper)`, with a
channel built as in [Quickstart — Wire the SDK](edge-sdk-quickstart.md#step-3b-wire-the-sdk).

---

## How Skill executions reach your adapter

| Concern | Where it lives |
| --- | --- |
| Receiving a command — typed (`takeOff`, `goTo`, ...) or custom (`mission.waypoint.execute`, ...) | `EdgeAdapterService` — see [Edge Adapter](edge-sdk-adapter.md#custom-commands) |
| Reporting a command's progress and outcome | Command execution events through `LiveDataService` — see [Edge Adapter — Custom Commands](edge-sdk-adapter.md#custom-commands) |
| Cancelling a running command | `cancelExecution(sn, externalExecutionId)` on `EdgeAdapterService` |
| Declaring which commands your adapter supports | `getCapabilities` on `EdgeAdapterService`, and the Skill Registry — see [Connector](edge-sdk-connector.md#capabilities) |
| Authoring Applications and Skills, and running them | The Admin Console and the **Client SDK** |

---

## Configuration

The Mission Autonomy address is `grpc.client.mission-autonomy.host` / `.port`
(`MISSION_AUTONOMY_SERVICE_HOST` / `MISSION_AUTONOMY_SERVICE_PORT`, default port `8004`). See the
[Configuration Guide](edge-sdk-configuration.md#grpc-client-configuration).
