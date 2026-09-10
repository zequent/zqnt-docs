# Edge SDK (Python) — Edge Adapter

Implementing an edge adapter in Python boils down to subclassing `EdgeAdapter` and overriding the methods your hardware supports. Methods you **don't** override automatically return `EdgeResponse.not_supported(...)` and are reported as unavailable in `get_capabilities`.

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
    metadata: dict[str, str]
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

`ok(...)` also accepts `external_execution_id` — set it when the command you just accepted keeps running asynchronously, so a later notification can be correlated back to it (see [Live Data](edge-sdk-python-live-data.md#notifications)).

For long-running commands you can stream multiple `EdgeResponse` objects via the streaming variants (see below).

---

## Method groups

The base class organises methods into groups. You only override what your hardware supports.

### Capability management (required)

| Method                                                         | Purpose                                                  |
|----------------------------------------------------------------|----------------------------------------------------------|
| `get_capabilities(sn, asset_id) -> Capabilities`               | Returns which commands the asset supports.               |
| `_auto_capabilities(sn, asset_type) -> Capabilities`           | Helper that introspects overridden methods.              |

### Flight control

| Method                                                                                | Notes                                |
|---------------------------------------------------------------------------------------|--------------------------------------|
| `take_off(ctx, coordinates: Coordinates) -> EdgeResponse`                             | Launch the asset.                    |
| `go_to(ctx, coordinates: Coordinates) -> EdgeResponse`                                | Fly to a target.                     |
| `return_to_home(ctx, request: ReturnToHomeRequest) -> EdgeResponse`                   | Trigger RTH.                         |

### Manual control

| Method                                                                                                     | Notes                                                            |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| `enter_manual_control(ctx, request: ManualControlRequest) -> EdgeResponse`                                 | Begin a manual session for a client.                             |
| `exit_manual_control(ctx, request: ManualControlRequest) -> EdgeResponse`                                  | Tear it down.                                                    |
| `manual_control_input(ctx, inputs: AsyncIterator[ManualControlInput]) -> EdgeResponse`                     | Stream of stick inputs. Iterate with `async for`.                |

### Dock and asset operations

| Method                                                       | Notes                                            |
|--------------------------------------------------------------|--------------------------------------------------|
| `open_cover(ctx) -> EdgeResponse`                            | Dock cover open.                                 |
| `close_cover(ctx, *, force: bool = False) -> EdgeResponse`   | Dock cover close.                                |
| `boot_sub_asset(ctx, *, value: bool) -> EdgeResponse`        | Power on/off a sub-asset (e.g. drone in dock).   |
| `change_ac_mode(ctx, *, mode: AssetAirConditionerState) -> EdgeResponse`     | Change air-conditioning mode.        |
| `debug_mode(ctx, *, value: bool) -> EdgeResponse`            | Enter/exit debug mode.                           |

### Camera

| Method                                                                       | Notes                       |
|------------------------------------------------------------------------------|-----------------------------|
| `change_lens(ctx, request: ChangeLensRequest) -> EdgeResponse`  | Switch lens.                |
| `change_zoom(ctx, request: ChangeZoomRequest) -> EdgeResponse`  | Set zoom factor.            |
| `look_at(ctx, coordinates: Coordinates) -> EdgeResponse`                     | Aim camera at a point.      |
| `capture_photo(ctx) -> EdgeResponse`                                         |                             |
| `start_recording(ctx) -> EdgeResponse`                                       |                             |
| `stop_recording(ctx) -> EdgeResponse`                                        |                             |

### Live streaming

| Method                                                                       | Notes                       |
|------------------------------------------------------------------------------|-----------------------------|
| `start_live_stream(ctx, request: LiveStreamStartRequest) -> EdgeResponse`    |                             |
| `stop_live_stream(ctx, request: LiveStreamStopRequest) -> EdgeResponse`      |                             |

### Tasks

| Method                                                  | Notes                                          |
|---------------------------------------------------------|------------------------------------------------|
| `prepare_task(ctx, task: Task) -> EdgeResponse`         | Validate, route, and stage a task.             |
| `start_task(ctx, task: Task) -> EdgeResponse`           | Begin execution.                               |
| `stop_task(ctx, task_id: str) -> EdgeResponse`          | Abort a running task.                          |

> **These receive only a task ID.** To act on one you must resolve it through the Connector and read
> the task's `WaypointTaskConfig` — the SAPIENT adapter implements them because its own protocol
> owns the task that ID refers to.
>
> The alternative is to leave them unimplemented (the `EdgeAdapter` default returns
> `NOT_IMPLEMENTED`) and accept `mission.waypoint.execute` through `send_custom_command` instead,
> where the waypoints and configuration arrive inline. That is what the MAVLink adapter does. Both
> are valid — pick one and document it, since a customer application has to know which to call.

---

## Reporting progress for long-running commands

There is no streaming variant of `EdgeResponse` for progress updates — `start_task` (and `send_custom_command` for vendor-specific commands) should return immediately with `EdgeResponse.ok(...)` and report progress separately via `LiveDataService.produce_notification(TaskEvent(...))`. See [Live Data — Notifications](edge-sdk-python-live-data.md#notifications).

```python
async def start_task(self, ctx: RequestContext, task_id: str) -> EdgeResponse:
    self._executor.submit(task_id)  # runs in the background, reports progress itself
    return EdgeResponse.ok(ctx.tid, ctx.sn, "Task started")

async def _on_progress(self, task_id: str, percent: float):
    await self._live.produce_notification(
        TaskEvent(
            task_id=task_id,
            task_type=TaskType.WAYPOINT,
            status=TaskStatus.RUNNING,
            sn=self._sn,
            progress=percent / 100,
        )
    )
```

The one real async-generator method on `EdgeAdapter` is `get_detections`, used for pull-based detection streaming (see [Method groups](#method-groups) above) — it is not a general progress-reporting mechanism.

---

## Best practices

- **Keep methods async-friendly.** Wrap blocking SDK calls with `asyncio.to_thread(...)` or use the vendor SDK's async API.
- **Always honour `ctx.tid` and `ctx.sn`** when constructing responses; the platform correlates by these.
- **Don't catch broad exceptions silently.** Convert known hardware errors to `EdgeResponse.fail(tid, sn, ErrorMessage(...))` with a meaningful `ErrorCode`; let the gRPC server surface the rest.
- **Don't keep state on the adapter** for per-request lifetimes. Use the `tid` as a key into a per-command map if you must.
- **Use `_auto_capabilities`** rather than maintaining the capability list manually.

---

## Testing your adapter

`EdgeServer` works in-process with a real `grpc.aio` server, so a typical test:

```python
import pytest
from edge_sdk import EdgeServer
from edge_sdk.generated import edge_pb2_grpc, common_pb2

@pytest.mark.asyncio
async def test_takeoff(monkeypatch):
    server = EdgeServer(adapter=MyDeviceAdapter(), port=0)
    addr = await server.start()
    # ... use a generated stub to call take_off and assert on the response
    await server.stop()
```

Or use the lower-level dispatcher with a `MagicMock` for purely unit-style tests; see the SDK's own `tests/` directory for examples.
