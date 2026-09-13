# Edge SDK (Python) — Mission Autonomy

`MissionAutonomyClient` is a small, focused client: it lets an edge adapter look up a **scheduler** definition directly. Everything else related to running automated behavior on an asset — receiving task lifecycle calls, reporting progress, custom commands — happens through other parts of the SDK, described below.

For Java, see [edge-sdk-mission-autonomy.md](edge-sdk-mission-autonomy.md).

---

## Scheduler lookup

```python
from edge_sdk import MissionAutonomyClient

client = MissionAutonomyClient(host="localhost", port=8004)
await client.connect()
try:
    scheduler = await client.get_scheduler(scheduler_id="scheduler-uuid")
finally:
    await client.close()
```

> **Beta preview — 2.0.x, not yet released.** This client's surface itself doesn't change on the
> unmerged branch (it's already just scheduler lookup today) — but the `SchedulerDTO` it returns
> does: `mission_id`/`task_id` are retired on the wire, replaced with a direct capability-execution
> target (`asset_sn` + `command_id`, or `asset_sn` + `application_id` + `skill_id`). This SDK's
> `edge_sdk.models.scheduler.SchedulerDTO` is its own plain-Python model, separate from (but
> field-equivalent to) the client SDK's own `SchedulerDTO` — both mirror the same
> `SchedulerProtoDTO` wire message. See the
> [Client SDK 2.0.x reference](../api-reference/client-sdk-mission-autonomy-python-2.0.md#schedulers--same-methods-different-schedulerdto-shape)
> for the full field breakdown.

---

## Receiving tasks (the common case)

The platform drives task execution by calling *into* your `EdgeAdapter` — you don't poll or manage tasks yourself. Task methods receive a `task_id`; fetch whatever your adapter actually needs (e.g. a stored flight plan) through `ConnectorClient` rather than through `MissionAutonomyClient`:

```python
from edge_sdk import EdgeAdapter, EdgeResponse, ErrorMessage, ErrorCode

class MyAdapter(EdgeAdapter):

    async def prepare_task(self, ctx, task_id: str) -> EdgeResponse:
        task = await self._connector.get_task(task_id, sn=ctx.sn)
        if task is None:
            return EdgeResponse.fail(ctx.tid, ctx.sn,
                ErrorMessage(message="No task found for this id", code=ErrorCode.CLIENT_ERROR))
        self._pending[task_id] = task
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Task prepared")

    async def start_task(self, ctx, task_id: str) -> EdgeResponse:
        self._executor.submit(task_id, self._pending[task_id])
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Task started")

    async def stop_task(self, ctx, task_id: str) -> EdgeResponse:
        await self._executor.cancel(task_id)
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Task stopped")
```

## Custom commands

For vendor-specific commands that don't map to a standard `EdgeAdapter` method (e.g. a proprietary waypoint-execute call), override `send_custom_command`:

```python
from edge_sdk import CustomCommandRequest, CustomCommandResponse

class MyAdapter(EdgeAdapter):

    async def send_custom_command(
        self, ctx, request: CustomCommandRequest,
    ) -> CustomCommandResponse:
        if request.command_type == "mission.waypoint.execute":
            execution_id = await self._start_waypoint_mission(request.params)
            return CustomCommandResponse.ok(
                ctx.tid, ctx.sn, request.command_type,
                external_execution_id=execution_id,  # lets mission-autonomy cancel/correlate later
            )
        return CustomCommandResponse.not_supported(ctx.tid, ctx.sn, request.command_type)
```

Set `external_execution_id` when the command you just accepted keeps running asynchronously — mission-autonomy uses it to later cancel the command (`StopTask`) and to correlate progress notifications back to this specific execution.

---

## Reporting progress

Progress flows back to the platform via `LiveDataService.produce_notification`, not as a task RPC return value — see [Live Data](edge-sdk-python-live-data.md).

---

## Where things live

| Concern | Where it lives |
| --- | --- |
| Receiving `prepare_task`/`start_task`/`stop_task` calls | `EdgeAdapter` — see [Edge Adapter](edge-sdk-python-adapter.md) |
| Vendor-specific commands | `EdgeAdapter.send_custom_command` (above) |
| Reporting progress/telemetry while a task runs | `LiveDataService` — see [Live Data](edge-sdk-python-live-data.md) |
| Declaring which commands your adapter supports | `get_capabilities` on `EdgeAdapter` — see [Connector](edge-sdk-python-connector.md#capabilities) |
| Creating missions and tasks, and triggering them | The **Client SDK**, used by customer applications |

---

## Best practices

> These apply if your adapter implements the task methods at all. SAPIENT does — but by passing
> `task_id` straight through as its own protocol's task identifier, not by resolving it through
> `ConnectorClient`: its `prepare_task` is a no-op acknowledgment, and `start_task`/`stop_task`
> forward `task_id` directly into a SAPIENT control command. MAVLink implements none of the task
> methods — it accepts `mission.waypoint.execute` through `send_custom_command` instead, with
> waypoints and configuration arriving inline, and none of this applies. See
> [Edge Adapter](edge-sdk-python-adapter.md#tasks).

- **Validate in `prepare_task`**; return an error there if you can't handle the task. Don't accept and then fail in `start_task`.
- **Make `start_task` non-blocking.** Schedule the work and return success immediately. Use `LiveDataService` to report state.
- **Idempotent `stop_task`.** Cancelling a task that's already finished must be a no-op.
- **Persist `task_id`** if you need to recover after a restart; the platform may re-issue a `start_task` for a task you already started.
