# Zequent Client SDK (Python) — Remote Control

`client.remote_control` sends direct, imperative commands to a connected asset — flight ops, dock
ops, and manual control. Every call targets a single asset by serial number (`sn`).

Full method-by-method reference: [Remote Control API Reference](../api-reference/client-sdk-remote-control-python.md).

For response semantics — what "success" actually means, and how command responses relate to
progress/telemetry — see [Functional Responses](FUNCTIONAL_RESPONSES_PYTHON.md).

For Java, see [REMOTE_CONTROL.md](REMOTE_CONTROL.md).

## Flight ops

```python
from client_sdk.models import TakeoffRequest

request = TakeoffRequest(sn="YOUR_DEVICE_SN", latitude=52.520008, longitude=13.404954, altitude=50.0)

response = await client.remote_control.takeoff(request)
if not response.success:
    print(f"Takeoff failed: {response.error.error_message}")
else:
    print(f"Takeoff accepted: {response.tid}")
```

`mission_id`/`task_id` on `TakeoffRequest`/`GoToRequest`/`ReturnToHomeRequest` are optional — set
them to correlate the command with a mission/task you already created via
[Mission Autonomy](MISSION_AUTONOMY_PYTHON.md).

## Manual control

`start_manual_control_input(sn)` returns a `ManualControlInputSession` — use it as `async with`,
call `send_input` per frame, then `complete()` to close the stream and get the final response.
`enter_manual_control`/`exit_manual_control` take/release exclusive control around the session:

```python
from client_sdk.models import ManualControlRequest, ManualControlInput

await client.remote_control.enter_manual_control(
    ManualControlRequest(sn=sn, client_id="app-1", user_id="user-1", session_id="sess-1")
)

async with client.remote_control.start_manual_control_input(sn) as session:
    await session.send_input(ManualControlInput(roll=0.1, pitch=0.0, yaw=0.0, throttle=0.5))
    response = await session.complete()

await client.remote_control.exit_manual_control(
    ManualControlRequest(sn=sn, client_id="app-1", user_id="user-1", session_id="sess-1")
)
```

## Dock and asset ops

All eight of these take a `DockOperationRequest` (`sn`, `asset_id`, optional `value: bool` whose
meaning differs per method — `force` for `close_cover`, `boot` for `boot_sub_asset`, `enabled` for
`debug_mode`, ignored otherwise); see the
[reference](../api-reference/client-sdk-remote-control-python.md#dock-and-asset-operations) for the
full list.

```python
from client_sdk.models import DockOperationRequest

request = DockOperationRequest(sn="YOUR_DOCK_SN", value=True)
response = await client.remote_control.debug_mode(request)
```

**`change_ac_mode` cannot actually change the AC mode** — confirmed in source: the underlying proto
request needs a `mode` field that `DockOperationRequest` has no way to set, so this method always
sends the idle mode no matter what you intend. This is a real, pre-existing SDK limitation, not a
documentation gap — there is currently no way to choose an AC mode from Python.

## No capability discovery, no custom commands

Unlike the Java (`getCapabilities`/`sendCustomCommand`) and Go (`GetCapabilities`/
`SendCustomCommand`) SDKs, **the Python Client SDK has neither** — confirmed against the real
source, there is no capability-related or custom-command method anywhere on `client.remote_control`.
If your application needs live capability discovery or vendor-defined custom commands, call it from
Java or Go, or query the platform some other way.

## Error handling

`RemoteControlResponse` uses `success`/`error.error_message` rather than raising for expected
business errors — a rejected command comes back as a normal, successfully-returned response with
`success=False`:

```python
response = await client.remote_control.go_to(request)
if not response.success:
    print(f"GoTo failed: {response.error.error_message}")
```

A transport failure (network down, deadline exceeded) still raises `grpc.aio.AioRpcError` —
`await` the call inside your own `try`/`except` if you need to handle that separately.

## See also

- [Remote Control API Reference](../api-reference/client-sdk-remote-control-python.md) — every method
- [Functional Responses](FUNCTIONAL_RESPONSES_PYTHON.md) — what a response actually confirms
