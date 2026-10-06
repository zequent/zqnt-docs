# Zequent Client SDK — Remote Control API Reference

Method reference for `client.remoteControl()`. For a narrative introduction and worked examples, see
the [Remote Control guide](../client-sdk/REMOTE_CONTROL.md). For Python, see
[client-sdk-remote-control-python.md](client-sdk-remote-control-python.md); for Go,
[client-sdk-remote-control-go.md](client-sdk-remote-control-go.md).

Every method targets one asset by serial number (`sn`) and returns a `CompletableFuture`, except
`startManualControlInput`. A client credential reaches only its own organization's assets; any
other serial number is refused with `PERMISSION_DENIED`.

## How a command reaches the asset

Every command on this page except manual control runs as a **single-command run** on the platform —
the same execution engine that runs Applications. The platform checks it (a route through a no-fly
zone is flown around it, a command with no possible detour is refused, a zone that requires approval
holds it until someone approves), dispatches it to the asset's edge adapter, and answers once the
adapter has replied.

- `success == false`: the command was refused before it reached the asset, or the adapter rejected it;
  `error.errorCode` / `error.errorMessage` say why.
- `success == true`: `progress.state` carries the run's status. `SKILL_EXECUTION_STATUS_SUCCEEDED` — the adapter carried
  out a command that completes at once. `SKILL_EXECUTION_STATUS_RUNNING` — the command is under
  way: `takeoff`, `goTo`, `lookAt` and `returnToHome` stay running until the adapter reports their outcome.

Neither confirms the physical outcome — see [Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md).

## Flight

| Method | Returns | Purpose |
| --- | --- | --- |
| `takeoff(TakeoffRequest)` | `TakeoffResponse` | Take off and fly to `latitude`/`longitude`/`altitude` (`flight.takeoff`) |
| `goTo(GoToRequest)` | `RemoteControlResponse` | Fly to `latitude`/`longitude`/`altitude` (`navigation.go_to`). The route is planned around no-fly zones. `altitude` is relative to the takeoff point |
| `goTo(GoToRequest, boolean noFlyZoneOverride)` | `RemoteControlResponse` | With `true`, fly straight through a no-fly zone that would otherwise refuse the fly-to. Honoured only for an organization admin or system admin, and recorded in the run's safety audit; refused for a client credential. `false` is the same as `goTo(request)` |
| `returnToHome(ReturnToHomeRequest)` | `RemoteControlResponse` | Return home (`flight.return_to_home`); `altitude` is optional, omit it for the device's default |
| `lookAt(LookAtRequest)` | `RemoteControlResponse` | Point the gimbal/camera at `latitude`/`longitude`/`altitude` (`gimbal.look_at`) |

Coordinates must be finite; latitude within ±90, longitude within ±180. The SDK throws
`IllegalArgumentException` otherwise.

## Manual control

Manual control is a live session with the asset and is not run as a command run.

| Method | Returns | Purpose |
| --- | --- | --- |
| `enterManualControl(ManualControlRequest)` | `RemoteControlResponse` | Take manual control. `clientId`, `userId` and `sessionId` are required; `reason` is optional |
| `exitManualControl(ManualControlRequest)` | `RemoteControlResponse` | Release manual control (same request shape) |
| `startManualControlInput(String sn, String assetId)` | `ManualControlInputSession` | Open the stick-input stream: call `sendInput(ManualControlInput)` per frame, then `complete()` for the final `RemoteControlResponse`. `completeWithError(Throwable)` and `close()` end it early |

## Dock, asset and camera

These take a `DockOperationRequest` (`sn`, optional `assetId`, optional `value`, `acMode`) and return
`RemoteControlResponse`.

| Method | Command | Request fields |
| --- | --- | --- |
| `openCover(DockOperationRequest)` | `dock.open_cover` | — |
| `closeCover(DockOperationRequest)` | `dock.close_cover` | `value`: `true` forces the close |
| `startCharging(DockOperationRequest)` | `dock.start_charging` | — |
| `stopCharging(DockOperationRequest)` | `dock.stop_charging` | — |
| `rebootAsset(DockOperationRequest)` | `asset.reboot` | — |
| `bootSubAsset(DockOperationRequest)` | `asset.boot_sub_asset` | `value`: `true` (or unset) powers the sub-asset on, `false` off |
| `debugMode(DockOperationRequest)` | `asset.remote_debug` | `value`: `true` enables remote debug mode, `false` (or unset) disables it |
| `changeAcMode(DockOperationRequest)` | `asset.change_ac_mode` | `acMode` (required): `IDLE`, `COOL`, `HEAT` or `DEHUMIDIFICATION` |
| `takePhoto(DockOperationRequest)` | `camera.take_photo` | — |
| `liveStreamSplitScreen(LiveStreamSplitScreenRequest)` | `stream.split_screen` | `sn`, optional `assetId`, `enabled` |

Lens and zoom changes are on Live Data — see [Live Video](../client-sdk/LIVE_VIDEO.md).

## Capabilities & custom commands

| Method | Returns | Purpose |
| --- | --- | --- |
| `getCapabilities(String sn)` | `CapabilitySnapshot` | What this asset supports right now, including vendor- and payload-specific commands that have no method on this page |
| `sendCustomCommand(CustomCommandRequest)` | `CustomCommandResponse` | Run any command by id, as a single-command run. Build the request from a descriptor with `CustomCommandRequest.forCapability(sn, descriptor, params)` |

`CapabilitySnapshot` carries `assetSn`, `assetType`, `revision`, `snapshotState` (e.g.
`CAPABILITY_SNAPSHOT_STATE_CURRENT`, or `..._STALE` when the asset is unreachable), `observedAt`,
`validUntil` and `capabilities`.

Each `CapabilityDescriptor` carries `commandId`, `displayName`, `description`, `state`,
`unavailableReason` (set when the command can't be used right now), `targetType` (`ASSET`,
`SUB_ASSET`, `PAYLOAD`, `COMPONENT`), `targetRef`, `schemaVersion`, `inputSchema`/`outputSchema`,
`constraints`, `metadata`, `errors`, `events`, `requirements`, `skillId`, `source` and `provider`.

`CustomCommandResponse` carries `success`, `commandType`, `error`, and either `result` (the command's
output, when the run already has it) or `progress` (the run's status).
