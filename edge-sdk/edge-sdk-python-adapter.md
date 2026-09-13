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

`ok(...)` also accepts `external_execution_id` — set it when the command you just accepted keeps running asynchronously, so a later notification can be correlated back to it (see [Live Data](edge-sdk-python-live-data.md#notifications)).

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
- **Tasks** — `prepare_task`, `start_task`, `stop_task` — each takes a bare `task_id: str`, not a `Task` object
- **Custom commands** — `send_custom_command`

```python
class MyDroneAdapter(EdgeAdapter):
    async def get_capabilities(self, sn, asset_id):
        return self._auto_capabilities(sn, AssetType.AIRCRAFT)

    async def take_off(self, ctx, coordinates):
        await hardware.take_off(coordinates.latitude, coordinates.longitude)
        return EdgeResponse.ok(ctx.tid, ctx.sn)
```

### Tasks

`prepare_task`/`start_task`/`stop_task` each receive only a bare `task_id: str` — not a `Task`
object. No confirmed real Python adapter resolves that ID through `ConnectorClient.get_task` (see
the [Connector reference](../api-reference/edge-sdk-python-connector-reference.md#missions-and-tasks)):
SAPIENT implements all three, but `prepare_task` is a no-op acknowledgment and `start_task`/
`stop_task` forward `task_id` straight into a SAPIENT protocol control command, with no Connector
lookup. MAVLink implements none of the three — it accepts `mission.waypoint.execute` through
`send_custom_command` instead, with waypoints and configuration arriving inline, so no task lookup
is needed at all. See
[Mission Autonomy — Best practices](edge-sdk-python-mission-autonomy.md#best-practices) for the full
per-adapter picture. Both approaches are valid; leaving the task methods unimplemented returns
`NOT_IMPLEMENTED`, which callers handle — pick one and document it.

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
