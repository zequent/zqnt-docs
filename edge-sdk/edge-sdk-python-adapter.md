# Edge SDK (Python) — Edge Adapter

Implementing an edge adapter in Python boils down to subclassing `EdgeAdapter` and overriding the methods your hardware supports. Methods you **don't** override automatically return `EdgeResponse.not_supported(...)` and are reported as unavailable in `get_capabilities`.

Full method-by-method reference, every real method and signature: [Edge Adapter API Reference](../api-reference/edge-sdk-python-adapter-reference.md).

For Java, see [edge-sdk-adapter.md](edge-sdk-adapter.md).

---

## The contract

```python
from edge_sdk import EdgeAdapter

class MyAdapter(EdgeAdapter):
    async def get_capabilities(self, sn: str, asset_id: str | None) -> Capabilities:
        ...
```

`get_capabilities` is the **only** abstract method. Use the helper:

```python
return self._auto_capabilities(sn, AssetType.DOCK)
```

`_auto_capabilities` introspects which methods you've overridden and produces a `Capabilities` payload that mirrors that exactly. You only maintain the implementation list — the capability list stays in sync automatically.

---

## RequestContext

Every command receives a `RequestContext` as the first argument:

```python
@dataclass
class RequestContext:
    tid: str        # transaction id (correlate response/progress to request)
    sn: str         # asset serial number
    timestamp: datetime
```

Always pass `ctx.tid` and `ctx.sn` back into your `EdgeResponse` to keep the platform's correlation working.

---

## EdgeResponse

The unified response type used by every unary command. Factories are named `ok`/`fail`, not `success`/`error`:

```python
EdgeResponse.ok(tid, sn, message="Takeoff initiated")
EdgeResponse.fail(tid, sn, ErrorMessage(message="Hardware fault", code=ErrorCode.ASSET_ERROR))
EdgeResponse.not_supported(tid, sn)            # default for un-overridden methods
EdgeResponse.ok(tid, sn, progress=CommandProgress(progress=42.0, state="climbing", left_time_seconds=30.0))
```

`ok(...)` also accepts `external_execution_id` — set it when the command you just accepted keeps running asynchronously, so its command execution events can be correlated back to it (see [Reporting progress](#reporting-progress-for-long-running-commands)).

For long-running commands you can stream multiple `EdgeResponse` objects via the streaming variants (see below).

---

## Method groups

The base class organises methods into groups — you only override what your hardware supports. See
the [reference](../api-reference/edge-sdk-python-adapter-reference.md) for every method's exact signature; the
categories are:

- **Capability management** (required) — `get_capabilities`, `_auto_capabilities`
- **Flight control** — `take_off`, `go_to`, `return_to_home`
- **Manual control** — `enter_manual_control`, `exit_manual_control`, `manual_control_input` (client-streaming)
- **Camera and gimbal** — `look_at`, `take_photo`, `capture_photo`, `change_lens`, `change_zoom`, `enable_gimbal_tracking`
- **Live streaming and recording** — `start_live_stream`, `stop_live_stream`, `start_recording`, `stop_recording`
- **Detection** — `get_detections` (the one server-streaming, async-generator method)
- **Dock and asset operations** — `open_cover`, `close_cover`, `start_charging`, `stop_charging`, `reboot_asset`, `boot_up_sub_asset`, `boot_down_sub_asset`, `register_asset`, `deregister_asset`
- **Debug and maintenance** — `enter_or_close_remote_debug_mode`, `change_ac_mode`
- **Tasks** — `prepare_task`, `start_task`, `stop_task` — see [Tasks](#tasks)
- **Custom commands** — `register_command`, and `send_custom_command`, which dispatches to what you registered

```python
class MyDroneAdapter(EdgeAdapter):
    async def get_capabilities(self, sn, asset_id):
        return self._auto_capabilities(sn, AssetType.AIRCRAFT)

    async def take_off(self, ctx, coordinates):
        await hardware.take_off(coordinates.latitude, coordinates.longitude)
        return EdgeResponse.ok(ctx.tid, ctx.sn)
```

### Custom commands

Declare each command your adapter runs with `register_command`. One registration does both halves:
the command appears in `_auto_capabilities`, and the default `send_custom_command` dispatches to its
handler. A well-known command (`mission.waypoint.execute`, ...) takes its description and input
schema from the platform's catalog, so registering it is one line; a hardware-specific command goes
under a `vendor.` prefix and brings its own `input_schema` (built with `schema(...)`). A handler takes
the `RequestContext` and the command's `params`, and returns a `CustomCommandResponse` — see the
example below.

### Tasks

`prepare_task`/`start_task`/`stop_task` still exist, and by default delegate to the registered
commands `mission.prepare`/`mission.start`/`mission.stop`. The 2.0 platform never calls `prepare_task`
or `start_task`. It calls `stop_task` to **cancel a running command**, passing that command's
`external_execution_id` as `task_id` — so implement `stop_task` if your long-running commands can be
aborted. See [Upgrading from 1.3](../concepts/migration-guide.md#task-based-execution-is-gone).

---

## Reporting progress for long-running commands

How a custom command's handler answers decides when the platform considers it finished — which is
when a Skill execution moves on to its next step. (`take_off`, `go_to`, `look_at` and `return_to_home`
always count as still running: report their outcome with events as below.)

- **Finished at once:** return `CustomCommandResponse.ok(ctx.tid, ctx.sn, command_id, result={...})`.
  The `result` is the step's output.
- **Still running:** return `ok(...)` with an `external_execution_id` and no `result`, and report the
  outcome as **command execution events** under that id: `RUNNING` with `progress` (0.0–1.0) while it
  runs, then exactly one of `SUCCEEDED` (optionally with `output`), `FAILED` or `CANCELLED`. Without a
  terminal event the Skill waits until it times out.

```python
import asyncio
import uuid

from edge_sdk import (
    AssetType, CommandExecutionEvent, CommandExecutionStatus, EdgeAdapter, LiveDataService, RequestContext,
)
from edge_sdk.models.common import CustomCommandResponse


class MyDroneAdapter(EdgeAdapter):

    def __init__(self, live: LiveDataService):
        super().__init__()
        self._live = live
        # One registration: advertised in get_capabilities, and dispatched by send_custom_command
        self.register_command("mission.waypoint.execute", self._fly_route)

    async def get_capabilities(self, sn, asset_id):
        return self._auto_capabilities(sn, AssetType.AIRCRAFT)

    async def _fly_route(self, ctx: RequestContext, params: dict) -> CustomCommandResponse:
        execution_id = str(uuid.uuid4())
        asyncio.create_task(self._fly(ctx.sn, execution_id, params["waypoints"]))
        # Accepted, not finished: the outcome follows as command execution events
        return CustomCommandResponse.ok(ctx.tid, ctx.sn, "mission.waypoint.execute",
                                        external_execution_id=execution_id)

    async def _fly(self, sn: str, execution_id: str, waypoints: list[dict]):
        for i, waypoint in enumerate(waypoints):
            await hardware.fly_to(waypoint)
            await self._report(sn, execution_id, CommandExecutionStatus.RUNNING, (i + 1) / len(waypoints))
        await self._report(sn, execution_id, CommandExecutionStatus.SUCCEEDED)

    async def _report(self, sn: str, execution_id: str, status: CommandExecutionStatus, progress=None):
        await self._live.produce_notification(CommandExecutionEvent(
            external_execution_id=execution_id,
            status=status,
            sn=sn,
            command_id="mission.waypoint.execute",
            progress=progress,
        ))
```

`produce_notification` is on `LiveDataService` (see [Live Data](edge-sdk-python-live-data.md#notifications));
2.0 no longer processes task events. The one real async-generator method on `EdgeAdapter` is
`get_detections`, used for pull-based detection streaming — it is not a progress-reporting mechanism.

---

## Best practices

- **Keep methods async-friendly.** Wrap blocking SDK calls with `asyncio.to_thread(...)` or use the vendor SDK's async API.
- **Always honour `ctx.tid` and `ctx.sn`** when constructing responses; the platform correlates by these.
- **Don't catch broad exceptions silently.** Convert known hardware errors to `EdgeResponse.fail(tid, sn, ErrorMessage(...))` with a meaningful `ErrorCode`; let the gRPC server surface the rest.
- **Don't keep state on the adapter** for per-request lifetimes. Use the `tid` as a key into a per-command map if you must.
- **Use `_auto_capabilities`** rather than maintaining the capability list manually.

---

## Testing your adapter

Adapter methods are plain coroutines, so a unit test calls them directly:

```python
from datetime import datetime

import pytest
from edge_sdk import Coordinates, RequestContext

from my_edge_adapter.adapter import MyDeviceAdapter


@pytest.mark.asyncio
async def test_takeoff():
    adapter = MyDeviceAdapter()
    ctx = RequestContext(tid="t-1", sn="DOCK-1", timestamp=datetime.now())

    response = await adapter.take_off(ctx, Coordinates(latitude=47.37, longitude=8.54, altitude=100.0))

    assert response.success
    assert response.tid == "t-1"
```

For an end-to-end check against a running adapter, see the `grpcurl` call in the
[Quickstart](edge-sdk-python-quickstart.md#step-6-verify-against-the-platform). The SDK's own `tests/`
directory has more examples.
