# Edge SDK (Python) — Edge Adapter API Reference

Exhaustive method reference for `EdgeAdapter`. For a narrative introduction and worked examples, see
the [Edge Adapter guide](../edge-sdk/edge-sdk-python-adapter.md). For Java, see
[edge-sdk-adapter.md](edge-sdk-adapter-reference.md).

Every method takes a `RequestContext` (`ctx`) as its first non-`self` argument and returns
`EdgeResponse`, except `get_detections` (an async-generator, server-streaming) and
`send_custom_command` (returns `CustomCommandResponse`). Methods you don't override return
`EdgeResponse.not_supported(...)` / `NotImplementedError` and are reported unavailable in
`get_capabilities`.

## RequestContext

```python
@dataclass
class RequestContext:
    tid: str        # transaction id (correlate response/progress to request)
    sn: str          # asset serial number
    timestamp: datetime
```

Exactly these three fields — no `metadata` field exists on `RequestContext`. Always pass `ctx.tid`
and `ctx.sn` back into your `EdgeResponse` to keep the platform's correlation working.

## EdgeResponse

```python
EdgeResponse.ok(tid, sn, message="Takeoff initiated")
EdgeResponse.fail(tid, sn, ErrorMessage(message="Hardware fault", code=ErrorCode.ASSET_ERROR))
EdgeResponse.not_supported(tid, sn)            # default for un-overridden methods
EdgeResponse.ok(tid, sn, progress=CommandProgress(progress=42.0, state="climbing", left_time_seconds=30.0))
```

`ok(...)` also accepts `external_execution_id` (keyword-only) — set it when the command you just
accepted keeps running asynchronously, so a later notification (or a `StopTask`) can be correlated
back to it. See [Live Data — Notifications](../edge-sdk/edge-sdk-python-live-data.md#notifications).

`Capability`/`Capabilities` field reference: [Models Reference — Capabilities](edge-sdk-python-models.md#capabilities).

## Capability management (required)

| Method | Returns | Notes |
| --- | --- | --- |
| `get_capabilities(sn, asset_id)` | `Capabilities` | The only abstract method |
| `_auto_capabilities(sn, asset_type)` | `Capabilities` | Helper — introspects which methods you've overridden; use this instead of building the list by hand |

## Flight control

| Method | Notes |
| --- | --- |
| `take_off(ctx, coordinates: Coordinates)` | Launch the asset |
| `go_to(ctx, coordinates: Coordinates)` | Fly to a target |
| `return_to_home(ctx, request: ReturnToHomeRequest)` | Trigger RTH |

## Manual control

| Method | Notes |
| --- | --- |
| `enter_manual_control(ctx, request: ManualControlRequest)` | Begin a manual session for a client |
| `exit_manual_control(ctx, request: ManualControlRequest)` | Tear it down |
| `manual_control_input(ctx, inputs: AsyncIterator[ManualControlInput])` | Client-streaming — iterate `inputs` with `async for` |

## Camera and gimbal

| Method | Notes |
| --- | --- |
| `look_at(ctx, coordinates: Coordinates, payload_index: str \| None, locked: bool \| None)` | Aim camera/gimbal at a point |
| `take_photo(ctx)` | Trigger a single photo capture |
| `capture_photo(ctx)` | Capture a photo and save it to storage — a distinct method from `take_photo`, both real |
| `change_lens(ctx, request: ChangeCameraLensRequest)` | Switch active lens |
| `change_zoom(ctx, request: ChangeCameraZoomRequest)` | Set zoom factor |
| `enable_gimbal_tracking(ctx, enabled: bool)` | Enable/disable gimbal auto-tracking |

## Live streaming and recording

| Method | Notes |
| --- | --- |
| `start_live_stream(ctx, request: LiveStreamStartRequest)` | Start a video live stream |
| `stop_live_stream(ctx, request: LiveStreamStopRequest)` | Stop the stream identified by `request.video_id` |
| `start_recording(ctx)` | Start video recording (separate from live streaming) |
| `stop_recording(ctx)` | Stop video recording |

## Detection (server-streaming)

| Method | Notes |
| --- | --- |
| `get_detections(ctx, stream_url: str \| None) -> AsyncIterator[DetectionResponse]` | The one real async-generator method on `EdgeAdapter` — implement it as `async def ... : yield DetectionResponse(...)`, not as a coroutine returning a value |

## Dock and asset operations

| Method | Notes |
| --- | --- |
| `open_cover(ctx)` | Open dock cover |
| `close_cover(ctx, force: bool \| None)` | Close dock cover; `force` is a required positional parameter, not keyword-only and has no default |
| `start_charging(ctx)` | Start charging the drone in the dock |
| `stop_charging(ctx)` | Stop charging |
| `reboot_asset(ctx)` | Reboot the main asset (dock) |
| `boot_up_sub_asset(ctx)` | Power on the sub-asset (drone) — two separate methods, not one `boot_sub_asset(value: bool)` |
| `boot_down_sub_asset(ctx)` | Power off the sub-asset |
| `register_asset(ctx, asset: Asset)` | **Not the same as `ConnectorClient.register_asset`.** This is the platform notifying your adapter that registration already happened — see [Connector reference](edge-sdk-python-connector-reference.md#assets) for the outbound call |
| `deregister_asset(ctx)` | Same distinction — notifies the adapter the asset was removed from the platform |

## Debug and maintenance

| Method | Notes |
| --- | --- |
| `enter_or_close_remote_debug_mode(ctx, enabled: bool)` | One toggle method, unlike the Java SDK's two separate `enterRemoteDebugMode`/`closeRemoteDebugMode` |
| `change_ac_mode(ctx, mode: AssetAirConditionerState)` | Change dock air-conditioner mode |

## Tasks

| Method | Notes |
| --- | --- |
| `prepare_task(ctx, task_id: str)` | Receives a bare `task_id` string, **not** a `Task` object |
| `start_task(ctx, task_id: str)` | Same — bare `task_id` string |
| `stop_task(ctx, task_id: str)` | Bare `task_id` string |

No confirmed real Python adapter resolves `task_id` through `ConnectorClient.get_task` — see
[Connector reference — Missions and tasks](edge-sdk-python-connector-reference.md#missions-and-tasks) and
[Mission Autonomy — Best practices](../edge-sdk/edge-sdk-python-mission-autonomy.md#best-practices)
for what SAPIENT and MAVLink actually do with these.

## Custom commands

| Method | Returns | Notes |
| --- | --- | --- |
| `send_custom_command(ctx, request: CustomCommandRequest)` | `CustomCommandResponse` | Handles a command identified by `request.command_type`, with parameters in `request.params: dict`. This is the path MAVLink and the simulator use for waypoint missions instead of the Task methods above |

## Capability-name mapping quirks

`_auto_capabilities` maps each overridable method name to a capability's `command` string via an
internal table. Two things worth knowing if you're cross-referencing against the Java SDK or the
platform's command IDs:

- `enter_or_close_remote_debug_mode` maps to a single `EnterOrCloseRemoteDebugMode` capability —
  there's no separate enter/close command pair the way the Java `EdgeAdapterService` has.
- The internal map also carries a `take_photo` → `TakePhoto` entry that is unrelated to (and
  independent of) the real, distinct `capture_photo` → `CapturePhoto` entry — both are real,
  separately-overridable methods; neither is a typo of the other.
