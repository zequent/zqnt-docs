# Zequent Client SDK (Go) — Remote Control API Reference

Exhaustive method reference for `remotecontrol.New(conn)`. For a narrative introduction, see the
[Quickstart](../client-sdk/QUICKSTART_GO.md#remotecontrol--manual-flightdock-control-gateway). For
Java, see [client-sdk-remote-control.md](client-sdk-remote-control.md).

Every method takes a `context.Context` first and returns `(*devicecontrol.CommandResponse, error)`
unless noted otherwise — there's no separate `HasErrors` flag to check; a non-nil `error` already
carries the platform-side message. Request/response types live in the `devicecontrol` package
(`device-control-contracts.proto`) — the same shapes `EdgeAdapterService` uses, so they flow
unchanged from this client through to the edge adapter that actually talks to the hardware.

## Capability / runtime discovery

| Method | Returns | Notes |
| --- | --- | --- |
| `ReportAssetRuntime(ctx, req)` | `*devicecontrol.ReportAssetRuntimeResponse` | Normally called by an edge adapter, not a customer app — exposed for completeness/testing |
| `GetAssetRuntime(ctx, assetSn, assetID)` | `*devicecontrol.AssetRuntimeSnapshot` | Latest payload/capability snapshot Remote Control has cached for an asset; `assetID` optional, pass `""` to omit |
| `GetCapabilities(ctx, sn)` | `*devicecontrol.AssetCapabilities` | Current capability snapshot for `sn` |

## Flight control

| Method | Notes |
| --- | --- |
| `TakeOff(ctx, sn, coordinate *devicecontrol.GeoCoordinate)` | |
| `GoTo(ctx, sn, coordinate *devicecontrol.GeoCoordinate)` | |
| `ReturnToHome(ctx, sn, altitude float32)` | `altitude` optional — pass `0` to omit and use the device's default RTH altitude |

## Manual control

| Method | Notes |
| --- | --- |
| `EnterManualControl(ctx, sn, clientID, userID, sessionID string)` | `clientID`/`userID` identify who is taking control; `sessionID` scopes the session |
| `ExitManualControl(ctx, sn, clientID, userID, sessionID string)` | Same parameters as `EnterManualControl` |
| `OpenManualControlInputStream(ctx) (grpc.ClientStreamingClient[devicecontrol.ManualControlInputCommandRequest, devicecontrol.CommandResponse], error)` | Client-streaming — send one `ManualControlInputCommandRequest` per tick, then `CloseAndRecv()` when done |
| `LookAt(ctx, sn, coordinate *devicecontrol.GeoCoordinate, locked bool)` | |
| `CapturePhoto(ctx, sn)` | |
| `PlayTTSAudio(ctx, req *devicecontrol.TextToSpeechCommandRequest)` | |
| `LiveStreamSplitScreen(ctx, sn, enabled bool)` | |

## Detection

| Method | Notes |
| --- | --- |
| `ControlDetection(ctx, req *devicecontrol.DetectionControlCommandRequest)` | |

## Dock commands

| Method | Notes |
| --- | --- |
| `OpenCover(ctx, sn)` | |
| `CloseCover(ctx, sn, force bool)` | `force` is a required parameter, not optional |
| `StartCharging(ctx, sn)` | |
| `StopCharging(ctx, sn)` | |

## Asset management

| Method | Notes |
| --- | --- |
| `RebootAsset(ctx, sn)` | |
| `BootSubAsset(ctx, sn, enabled bool)` | Power the sub-asset on/off |

## Debug and maintenance

| Method | Notes |
| --- | --- |
| `SetRemoteDebugMode(ctx, sn, enabled bool)` | One toggle method, unlike Java's two separate `enterRemoteDebugMode`/`closeRemoteDebugMode` |
| `ChangeAcMode(ctx, req *devicecontrol.ChangeAcModeCommandRequest)` | |

## Camera

| Method | Notes |
| --- | --- |
| `ChangeLens(ctx, req *devicecontrol.ChangeCameraLensCommandRequest)` | |
| `ChangeZoom(ctx, req *devicecontrol.ChangeCameraZoomCommandRequest)` | |

## Custom commands

| Method | Returns | Notes |
| --- | --- | --- |
| `SendCustomCommand(ctx, sn, commandID string, params *structpb.Struct, target *devicecontrol.CapabilityTarget)` | `*devicecontrol.CustomCommandResponse` | Handles a command that doesn't map to a standard method above — see [Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md) for the `mission.waypoint.execute` use case |
