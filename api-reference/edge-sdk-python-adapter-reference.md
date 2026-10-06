# Edge SDK (Python) — Edge Adapter API Reference

> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Edge Adapter reference](edge-sdk-python-adapter-reference-1.3.md).

Method reference for `EdgeAdapter`, the class your adapter extends. For a narrative introduction and
worked examples, see the [Edge Adapter guide](../edge-sdk/edge-sdk-python-adapter.md). For Java, see
[edge-sdk-adapter-reference.md](edge-sdk-adapter-reference.md).

Only `get_capabilities` is required. Every other method has a default; a method you neither
override nor back with `register_command` answers "not supported" and is reported `UNSUPPORTED` in
`_auto_capabilities`.

Every command method takes a `RequestContext` (`ctx`: `tid`, `sn`, `timestamp`) first and returns
`EdgeResponse`, except `send_custom_command` (`CustomCommandResponse`), `get_detections` (an async
generator) and `manual_control_input` (consumes an async iterator). Pass `ctx.tid` and `ctx.sn` back
into your response.

## How commands arrive

The platform runs work as Skills. Each command node of a run is dispatched to your adapter by its
command id: a built-in id lands on the matching typed method below, any other id on
`send_custom_command`.

- **Waits for your report.** `flight.takeoff`, `navigation.go_to`, `gimbal.look_at` and
  `flight.return_to_home`, and a custom command that returns no `result`, are *accepted*: the node
  stays running until you report a command execution event for it. Return
  `EdgeResponse.ok(..., external_execution_id=...)` (or `CustomCommandResponse.ok(...,
  external_execution_id=...)`) with your own execution id; without one, the platform uses
  `ctx.tid`. See [Edge Adapter guide](../edge-sdk/edge-sdk-python-adapter.md) and
  [Live Data — Notifications](../edge-sdk/edge-sdk-python-live-data.md#notifications).
- **Done on success.** Every other built-in command is complete when you return `ok`.
- **Cancel.** The platform cancels a running command by calling `stop_task(ctx, task_id)` with
  your execution id as `task_id`.

## Capabilities and command registration

| Method | Returns | Notes |
| --- | --- | --- |
| `get_capabilities(sn, asset_id)` | `Capabilities` | Required. Usually `return self._auto_capabilities(sn, AssetType.AIRCRAFT)` |
| `_auto_capabilities(sn, asset_type)` | `Capabilities` | Builds the snapshot from the platform's command catalog — `AVAILABLE` for each typed method you override, `UNSUPPORTED` otherwise — plus everything you registered |
| `register_command(command_id, handler=None, *, description, display_name, input_schema, output_schema, schema_version, target, skill_id, state, unavailable_reason, metadata)` | `None` | Declare a command: it is advertised by `_auto_capabilities` and, with a `handler`, run by the default `send_custom_command`. Catalog commands (e.g. `mission.waypoint.execute`) take their description and schemas from the catalog; your own commands go under a `vendor.` prefix with their own contract. Registering an id again replaces it (e.g. to mark it `TEMPORARILY_UNAVAILABLE`) |
| `registered_commands()` | `dict[str, RegisteredCommand]` | What you have registered (useful in tests) |

A handler is `async def handler(ctx, params: dict) -> CustomCommandResponse`. A typed method you do
not override runs the handler registered for its command id, so you can implement a built-in
command either way.

## Flight

| Method | Command id | Notes |
| --- | --- | --- |
| `take_off(ctx, coordinates: Coordinates)` | `flight.takeoff` | |
| `go_to(ctx, coordinates: Coordinates)` | `navigation.go_to` | Altitude is relative to the takeoff point |
| `return_to_home(ctx, request: ReturnToHomeRequest)` | `flight.return_to_home` | |
| `enter_manual_control(ctx, request: ManualControlRequest)` | `flight.manual.enter` | |
| `exit_manual_control(ctx, request: ManualControlRequest)` | `flight.manual.exit` | |
| `manual_control_input(ctx, inputs: AsyncIterator[ManualControlInput])` | — | Stick input stream; iterate `inputs` with `async for` |

## Camera, gimbal and stream

| Method | Command id | Notes |
| --- | --- | --- |
| `look_at(ctx, coordinates, payload_index, locked)` | `gimbal.look_at` | |
| `enable_gimbal_tracking(ctx, enabled: bool)` | `gimbal.tracking` | |
| `capture_photo(ctx)` | `camera.take_photo` | What the platform calls to take a photo |
| `take_photo(ctx)` | — | Separate typed call; the platform's photo command uses `capture_photo` |
| `change_lens(ctx, request: ChangeCameraLensRequest)` | `camera.change_lens` | |
| `change_zoom(ctx, request: ChangeCameraZoomRequest)` | `camera.change_zoom` | |
| `start_recording(ctx)` / `stop_recording(ctx)` | `camera.start_recording` / `camera.stop_recording` | Video recording |
| `start_live_stream(ctx, request: LiveStreamStartRequest)` | `stream.start` | Publish video to the stream server URL in the request |
| `stop_live_stream(ctx, request: LiveStreamStopRequest)` | `stream.stop` | Stop the stream `request.video_id` |
| `get_detections(ctx, stream_url) -> AsyncIterator[DetectionResponse]` | — | Server-streaming: write it as an async generator that `yield`s `DetectionResponse` |

## Dock and asset

| Method | Command id | Notes |
| --- | --- | --- |
| `open_cover(ctx)` | `dock.open_cover` | |
| `close_cover(ctx, force: bool \| None)` | `dock.close_cover` | |
| `start_charging(ctx)` / `stop_charging(ctx)` | `dock.start_charging` / `dock.stop_charging` | |
| `reboot_asset(ctx)` | `asset.reboot` | |
| `boot_up_sub_asset(ctx)` / `boot_down_sub_asset(ctx)` | `asset.boot_sub_asset` | Power the sub-asset (drone) on / off |
| `enter_or_close_remote_debug_mode(ctx, enabled: bool)` | `asset.remote_debug` | |
| `change_ac_mode(ctx, mode: AssetAirConditionerState)` | `asset.change_ac_mode` | |
| `register_asset(ctx, asset)` / `deregister_asset(ctx)` | — | Lifecycle callbacks when the platform adds or removes the asset; not commands. Pairing your asset is `ConnectorClient.ensure_asset` — see the [Connector reference](edge-sdk-python-connector-reference.md) |

## Custom commands and cancellation

| Method | Returns | Notes |
| --- | --- | --- |
| `send_custom_command(ctx, request: CustomCommandRequest)` | `CustomCommandResponse` | Every command id without a typed method (`request.command_type`, parameters in `request.params`). The default runs the handler you registered for it. Override it only to route ids dynamically, and call `await super().send_custom_command(ctx, request)` first |
| `stop_task(ctx, task_id)` | `EdgeResponse` | Cancel a running command; `task_id` is your execution id. By default it runs the handler registered for `mission.stop` |

## Responses

```python
EdgeResponse.ok(ctx.tid, ctx.sn, message="Cover opened")
EdgeResponse.ok(ctx.tid, ctx.sn, external_execution_id=run_id)   # accepted, still running
EdgeResponse.fail(ctx.tid, ctx.sn, ErrorMessage(message="Cover is blocked", code=ErrorCode.ASSET_ERROR))
EdgeResponse.not_supported(ctx.tid, ctx.sn)

CustomCommandResponse.ok(ctx.tid, ctx.sn, command_type, result={"photos": 12})   # done, with output
CustomCommandResponse.ok(ctx.tid, ctx.sn, command_type, external_execution_id=run_id)
CustomCommandResponse.fail(ctx.tid, ctx.sn, command_type, ErrorMessage(...))
```

`Capability`/`Capabilities` fields: [Models Reference](edge-sdk-python-models.md).
