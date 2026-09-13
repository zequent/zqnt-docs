# Zequent Client SDK (Python) — Remote Control API Reference

Exhaustive method reference for `client.remote_control`. For a narrative introduction, see the
[Remote Control guide](../client-sdk/REMOTE_CONTROL_PYTHON.md). For Java, see
[client-sdk-remote-control.md](client-sdk-remote-control.md); for Go,
[client-sdk-remote-control-go.md](client-sdk-remote-control-go.md).

Every method is a coroutine returning `RemoteControlResponse` (`success`/`error`/`message` — see
[Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES_PYTHON.md)) unless noted otherwise.

**No `get_capabilities` and no `send_custom_command`.** Confirmed against the real source — neither
exists anywhere in the Python Client SDK, unlike Java (`getCapabilities`/`sendCustomCommand`) and Go
(`GetCapabilities`/`SendCustomCommand`). There is currently no capability-discovery or custom-command
path from Python.

## Flight control

| Method | Notes |
| --- | --- |
| `takeoff(TakeoffRequest)` | `sn`, `latitude`, `longitude`, `altitude` required; `asset_id`/`mission_id`/`task_id` optional |
| `go_to(GoToRequest)` | Same fields as `TakeoffRequest` |
| `return_to_home(ReturnToHomeRequest)` | `altitude` optional — omit for the device's default RTH altitude |
| `look_at(LookAtRequest)` | `sn`, `latitude`, `longitude`, `altitude` required; `asset_id` optional |

## Manual control

| Method | Notes |
| --- | --- |
| `enter_manual_control(ManualControlRequest)` | `sn`, `client_id`, `user_id`, `session_id` required; `reason` optional |
| `exit_manual_control(ManualControlRequest)` | Same request shape as `enter_manual_control` |
| `start_manual_control_input(sn: str) -> ManualControlInputSession` | Opens the client-streaming RC-input channel — use as `async with`: `session.send_input(input)` per frame, then `await session.complete()` to get the final `RemoteControlResponse`. `session.complete_with_error(exc)` and `session.close()` are also available |

## Dock and asset operations

All eight of these take a `DockOperationRequest` (`sn`, `asset_id`, and an optional `value: bool`
whose meaning depends on the method — see the table):

| Method | `value` meaning | Notes |
| --- | --- | --- |
| `open_cover(DockOperationRequest)` | ignored | |
| `close_cover(DockOperationRequest)` | `force` | |
| `start_charging(DockOperationRequest)` | ignored | |
| `stop_charging(DockOperationRequest)` | ignored | |
| `reboot_asset(DockOperationRequest)` | ignored | |
| `boot_sub_asset(DockOperationRequest)` | `boot` (power on/off) | |
| `debug_mode(DockOperationRequest)` | `enabled` | Calls the `SetRemoteDebugMode` RPC — a single toggle, matching Go's `SetRemoteDebugMode`, unlike Java's two separate `enterRemoteDebugMode`/`closeRemoteDebugMode` |
| `change_ac_mode(DockOperationRequest)` | ignored | **Cannot actually change the AC mode.** Confirmed in source: the underlying proto request requires a `mode` field that `DockOperationRequest` has no way to set, so this method always sends `AIR_CONDITIONER_IDLE` regardless of intent — a known, pre-existing SDK limitation, not a documentation gap |

`value` defaults to `False`/ignored when omitted (`None`).
