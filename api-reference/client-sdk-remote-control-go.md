# Zequent Client SDK (Go) — Remote Control API Reference

Method reference for `remotecontrol.New(conn)` (package
`github.com/Zequent/zqnt-client-sdk-go/v2/remotecontrol`; dial remote-control-service, default port
8002). For a narrative introduction, see the
[Quickstart](../client-sdk/QUICKSTART_GO.md#remotecontrol--manual-flightdock-control-gateway). For
Java, see [client-sdk-remote-control.md](client-sdk-remote-control.md); for Python,
[client-sdk-remote-control-python.md](client-sdk-remote-control-python.md).

Every method takes a `context.Context` first and returns `(*devicecontrol.CommandResponse, error)`
unless noted otherwise. Request and response types are in
`github.com/Zequent/zqnt-client-sdk-go/v2/gen/devicecontrol/contracts/proto` (`devicecontrol`). There
is no separate `HasErrors` flag to check: a platform-side error comes back as a non-nil `error`
carrying the platform's message. A client credential reaches only its own organization's assets;
any other serial number is refused with `codes.PermissionDenied` (`status.Code(err)`).

## How a command reaches the asset

Every command on this page except manual control runs as a **single-command run** on the platform —
the same execution engine that runs Applications. The platform checks it (a route through a no-fly
zone is flown around it, a command with no possible detour is refused, a zone that requires approval
holds it until someone approves), dispatches it to the asset's edge adapter, and answers once the
adapter has replied.

- `err != nil`: the command was refused before it reached the asset, or the adapter rejected it;
  the error carries the platform's message.
- `err == nil`: `resp.GetProgress().GetState()` carries the run's status. `SKILL_EXECUTION_STATUS_SUCCEEDED` — the adapter carried
  out a command that completes at once. `SKILL_EXECUTION_STATUS_RUNNING` — the command is under
  way: `TakeOff`, `GoTo`, `LookAt` and `ReturnToHome` stay running until the adapter reports their outcome.

Neither confirms the physical outcome — see [Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES_GO.md).

## Capabilities and runtime

| Method | Returns | Notes |
| --- | --- | --- |
| `GetCapabilities(ctx, sn)` | `*devicecontrol.AssetCapabilities` | What this asset supports right now, including vendor- and payload-specific commands that have no method on this page |
| `GetAssetRuntime(ctx, assetSn, assetID)` | `*devicecontrol.AssetRuntimeSnapshot` | The asset's latest runtime snapshot: detected payloads, capabilities, revision, snapshot state. `assetID` is optional, pass `""` |

## Flight

| Method | Notes |
| --- | --- |
| `TakeOff(ctx, sn, coordinate *devicecontrol.GeoCoordinate)` | `flight.takeoff` |
| `GoTo(ctx, sn, coordinate *devicecontrol.GeoCoordinate)` | `navigation.go_to`. The route is planned around no-fly zones. `Altitude` is relative to the takeoff point |
| `GoToWithOptions(ctx, sn, coordinate, remotecontrol.GoToOptions{NoFlyZoneOverride: true})` | Flies straight through a no-fly zone that would otherwise refuse the fly-to. Honoured only for an organization admin or system admin; refused for a client credential |
| `ReturnToHome(ctx, sn, altitude float32)` | `flight.return_to_home`. Pass `0` for the device's default altitude |
| `LookAt(ctx, sn, coordinate *devicecontrol.GeoCoordinate, locked bool)` | `gimbal.look_at` |

## Manual control

Manual control is a live session with the asset and is not run as a command run.

| Method | Notes |
| --- | --- |
| `EnterManualControl(ctx, sn, clientID, userID, sessionID string)` | `clientID`/`userID` identify who takes control; `sessionID` scopes the session |
| `ExitManualControl(ctx, sn, clientID, userID, sessionID string)` | Same parameters |
| `OpenManualControlInputStream(ctx)` | Returns `grpc.ClientStreamingClient[devicecontrol.ManualControlInputCommandRequest, devicecontrol.CommandResponse]`. `Send` one `ManualControlInputCommandRequest` (with `Base.Sn` set) per frame, then `CloseAndRecv()` |

## Dock and asset

| Method | Command |
| --- | --- |
| `OpenCover(ctx, sn)` | `dock.open_cover` |
| `CloseCover(ctx, sn, force bool)` | `dock.close_cover` |
| `StartCharging(ctx, sn)` | `dock.start_charging` |
| `StopCharging(ctx, sn)` | `dock.stop_charging` |
| `RebootAsset(ctx, sn)` | `asset.reboot` |
| `BootSubAsset(ctx, sn, enabled bool)` | `asset.boot_sub_asset` — power the sub-asset on or off |
| `SetRemoteDebugMode(ctx, sn, enabled bool)` | `asset.remote_debug` |
| `ChangeAcMode(ctx, req *devicecontrol.ChangeAcModeCommandRequest)` | `asset.change_ac_mode` — set `Base` and `Mode` |

## Camera and stream

| Method | Command |
| --- | --- |
| `CapturePhoto(ctx, sn)` | `camera.take_photo` |
| `ChangeLens(ctx, req *devicecontrol.ChangeCameraLensCommandRequest)` | `camera.change_lens` — set `Base` and `Request.Lens` |
| `ChangeZoom(ctx, req *devicecontrol.ChangeCameraZoomCommandRequest)` | `camera.change_zoom` — set `Base`, `Request.Lens` and `Request.Zoom` |
| `LiveStreamSplitScreen(ctx, sn, enabled bool)` | `stream.split_screen` |

## Custom commands

| Method | Returns | Notes |
| --- | --- | --- |
| `SendCustomCommand(ctx, sn, commandID string, params *structpb.Struct, target *devicecontrol.CapabilityTarget)` | `*devicecontrol.CustomCommandResponse` | Run any command the asset publishes as a capability (vendor- or payload-specific ones, e.g. `speaker.audio.play`), as a single-command run. `GetCapabilities` lists the command ids and each one's parameter schema. The response carries `Result` (the command's output, when the run already has it) or `Progress` |
