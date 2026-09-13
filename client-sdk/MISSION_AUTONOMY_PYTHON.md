# Zequent Client SDK (Python) — Mission Autonomy

`client.mission_autonomy` creates and manages missions, waypoint tasks, and schedulers, and is the
only interface that can start, stop, pause, or resume a task. Unlike the Java SDK, `client.connector`
has no Mission/Task methods at all here — there's no "which one should I call" question in Python.

Full method-by-method reference: [Mission Autonomy API Reference](../api-reference/client-sdk-mission-autonomy-python.md).

For Java, see [CONNECTOR.md](CONNECTOR.md#missions-and-tasks-are-records-not-flights) /
[the Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy.md).

> **Beta preview — 2.0.x, not yet released.** An unmerged branch replaces everything on this page
> with an Application → Skill → SkillExecution model — every method below raises
> `LegacyOperationRemovedError` on that branch, since there's no backend RPC left for any of them.
> None of this is on `main`/the current 1.3.x release yet. See
> [Applications & Skills](../concepts/applications-and-skills-2.0.md) for the model that replaces
> this page, and the
> [2.0.x Beta reference](../api-reference/client-sdk-mission-autonomy-python-2.0.md) for the exact
> method-by-method reference.

## Creating a mission

```python
from client_sdk.models import MissionDTO, MissionType

mission = MissionDTO(name="North Perimeter Patrol", type=MissionType.PERIMETER_PATROL)

response = await client.mission_autonomy.create_mission(mission)
if not response.success:
    print(f"Create mission failed: {response.error.error_message}")
else:
    print(f"Mission created: {response.mission_id}")
```

`create_mission`/`update_mission` are **route-optimized** — confirmed against the backend
`MissionAutonomyGrpcService`: the mission is run through `missionRouteOptimizer.optimize(...)` before
being written. `create_task`/`update_task` get the equivalent no-fly-zone-aware expansion for a
waypoint task with a `mission_id`. `get_*`/`delete_*` are plain passthroughs — no optimization
applies to reads or deletes.

## Creating a waypoint task

```python
from client_sdk.models import TaskDTO, TaskType

task = TaskDTO(mission_id=mission_id, task_type=TaskType.WAYPOINT, name="Loop A")

response = await client.mission_autonomy.create_task(task)
```

## Task execution lifecycle

```python
await client.mission_autonomy.start_task(task_id)
await client.mission_autonomy.pause_task(task_id)
await client.mission_autonomy.resume_task(task_id)
await client.mission_autonomy.stop_task(task_id)
```

These forward a bare task ID to the adapter — works only where the adapter implements the task
lifecycle. See [Waypoint Missions](WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use) for the
full per-adapter picture, including the command-based alternative (MAVLink, the simulator) that
doesn't use this lifecycle at all.

## Schedulers

```python
from client_sdk.models import SchedulerDTO

scheduler = SchedulerDTO(name="Nightly patrol", cron_expression="0 22 * * *", mission_id=mission_id)
response = await client.mission_autonomy.create_scheduler(scheduler)

all_for_task = await client.mission_autonomy.list_schedulers(task_id)
everything = await client.mission_autonomy.list_schedulers()  # task_id omitted -- unfiltered
```

Identical scheduler methods also exist on `client.connector` — same wire messages, no optimization
difference (scheduler operations aren't route-related), so it makes no functional difference which
one you call.

## Error handling

Every method here returns a response object (`MissionResponse`/`TaskResponse`/`SchedulerResponse`)
with `success: bool` and `error: ErrorInfo | None` — **none of them raise for a business-level
error**, unlike `client.connector`'s asset/payload/organization/policy methods (which raise
`ConnectorError`; see [Connector — Error handling](CONNECTOR_PYTHON.md#error-handling)). Only a
transport failure raises, as `grpc.aio.AioRpcError`:

```python
import grpc

try:
    response = await client.mission_autonomy.create_mission(mission)
except grpc.aio.AioRpcError as e:
    # Transport failure -- couldn't reach the platform at all.
    raise
else:
    if not response.success:
        # Platform-side rejection, e.g. validation failure.
        print(response.error.error_message)
```

See [Functional Responses](FUNCTIONAL_RESPONSES_PYTHON.md) for what `success` actually confirms.

## See also

- [Mission Autonomy API Reference](../api-reference/client-sdk-mission-autonomy-python.md) — every method
- [Waypoint Missions](WAYPOINT_MISSIONS.md) — which adapter uses which execution path
- [Connector](CONNECTOR_PYTHON.md) — assets, organizations, technical config
