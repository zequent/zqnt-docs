# Zequent Client SDK (Python) — Remote Control API Reference

Method reference for `client.remote_control`. For a narrative introduction, see the
[Remote Control guide](../client-sdk/REMOTE_CONTROL_PYTHON.md). For Java, see
[client-sdk-remote-control.md](client-sdk-remote-control.md); for Go,
[client-sdk-remote-control-go.md](client-sdk-remote-control-go.md).

Every method is a coroutine that targets one asset by serial number (`sn`) and returns a
`RemoteControlResponse` (`success`, `tid`, `sn`, `asset_id`, `message`, `error`, `progress`), except
`start_manual_control_input`. A client credential reaches only its own organization's assets; any
other serial number raises `client_sdk.auth.ZequentAuthError` with `PERMISSION_DENIED`.

## How a command reaches the asset

Every command on this page except manual control runs as a **single-command run** on the platform —
the same execution engine that runs Applications. The platform checks it (a route through a no-fly
zone is flown around it, a command with no possible detour is refused, a zone that requires approval
holds it until someone approves), dispatches it to the asset's edge adapter, and answers once the
adapter has replied.

- `success` is `False`: the command was refused before it reached the asset, or the adapter rejected it;
  `error.error_message` says why.
- `success` is `True`: `progress.state` carries the run's status. `SKILL_EXECUTION_STATUS_SUCCEEDED` — the adapter carried
  out a command that completes at once. `SKILL_EXECUTION_STATUS_RUNNING` — the command is under
  way: `takeoff`, `go_to`, `look_at` and `return_to_home` stay running until the adapter reports their outcome.

Neither confirms the physical outcome — see [Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES_PYTHON.md).

## Flight

| Method | Request fields |
| --- | --- |
| `takeoff(TakeoffRequest)` | `sn`, `latitude`, `longitude`, `altitude`; optional `asset_id` |
| `go_to(GoToRequest, *, no_fly_zone_override=False)` | Same fields as `TakeoffRequest`. `altitude` is relative to the takeoff point. `no_fly_zone_override=True` flies straight through a no-fly zone that would otherwise refuse the fly-to; it is honoured only for an organization admin or system admin and refused for a client credential |
| `return_to_home(ReturnToHomeRequest)` | `sn`; optional `asset_id`, `altitude` (omit for the device's default) |
| `look_at(LookAtRequest)` | `sn`, `latitude`, `longitude`, `altitude`; optional `asset_id` |

## Manual control

Manual control is a live session with the asset and is not run as a command run.

| Method | Notes |
| --- | --- |
| `enter_manual_control(ManualControlRequest)` | `sn`, `client_id`, `user_id`, `session_id` required; `asset_id`, `reason` optional |
| `exit_manual_control(ManualControlRequest)` | Same request shape |
| `start_manual_control_input(sn) -> ManualControlInputSession` | The stick-input stream. Use it as `async with`: `await session.send_input(ManualControlInput(...))` per frame, then `await session.complete()` for the final `RemoteControlResponse`. `complete_with_error(exc)` and `close()` end it early |

## Dock and asset

These take a `DockOperationRequest` (`sn`, optional `asset_id`, optional `value: bool`).

| Method | Command | `value` |
| --- | --- | --- |
| `open_cover(DockOperationRequest)` | `dock.open_cover` | — |
| `close_cover(DockOperationRequest)` | `dock.close_cover` | `True` forces the close |
| `start_charging(DockOperationRequest)` | `dock.start_charging` | — |
| `stop_charging(DockOperationRequest)` | `dock.stop_charging` | — |
| `reboot_asset(DockOperationRequest)` | `asset.reboot` | — |
| `boot_sub_asset(DockOperationRequest)` | `asset.boot_sub_asset` | `True` powers the sub-asset on, `False` off |
| `debug_mode(DockOperationRequest)` | `asset.remote_debug` | `True` enables remote debug mode, `False` disables it |

## Capabilities and other commands

The Python client has no capability lookup and no custom-command call. To read what an asset
supports, or to run a command that has no method here (camera, payload or vendor-specific commands),
use the Java (`getCapabilities` / `sendCustomCommand`) or Go (`GetCapabilities` / `SendCustomCommand`)
client SDK, or put the command into a Skill — see
[Applications & Skills](../concepts/applications-and-skills.md).
