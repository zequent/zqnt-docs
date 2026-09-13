# Zequent Client SDK — Remote Control API Reference

Exhaustive method reference for `client.remoteControl()`. For a narrative introduction, response
semantics, and worked examples, see the [Remote Control guide](../client-sdk/REMOTE_CONTROL.md).

Every method targets a single asset by serial number (`sn`), and returns a `CompletableFuture`.
`RemoteControlResponse`/`TakeoffResponse`/`CustomCommandResponse` all use `isSuccess()` +
`getError().getErrorMessage()` — see
[Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md) for what `success` actually confirms.

## Flight ops

| Method | Returns | Purpose |
| --- | --- | --- |
| `takeoff(TakeoffRequest)` | `TakeoffResponse` | Take off and fly to a target lat/lon/altitude |
| `goTo(GoToRequest)` | `RemoteControlResponse` | Fly to a target lat/lon/altitude |
| `returnToHome(ReturnToHomeRequest)` | `RemoteControlResponse` | Return to home, optionally at a given altitude |
| `lookAt(LookAtRequest)` | `RemoteControlResponse` | Point the gimbal/camera at a lat/lon/altitude |

`missionId`/`taskId` on `TakeoffRequest`/`GoToRequest` are optional — set them to correlate the
command with a mission/task you already created via
[Connector](../client-sdk/CONNECTOR.md#missions-and-tasks-are-records-not-flights).

## Manual control

| Method | Returns | Purpose |
| --- | --- | --- |
| `enterManualControl(ManualControlRequest)` | `RemoteControlResponse` | Take exclusive manual control of an asset |
| `exitManualControl(ManualControlRequest)` | `RemoteControlResponse` | Release manual control |
| `startManualControlInput(sn, assetId)` | `ManualControlInputSession` | Open a gRPC streaming session for continuous stick input — not a `CompletableFuture`; call `sendInput` repeatedly, then `complete()` |

## Dock & asset ops

All of these take a `DockOperationRequest` (`sn`, `assetId`, optional `value`) except
`liveStreamSplitScreen`, and all return `RemoteControlResponse`.

| Method | `value` meaning |
| --- | --- |
| `openCover(DockOperationRequest)` | ignored |
| `closeCover(DockOperationRequest)` | `true` forces the close |
| `startCharging(DockOperationRequest)` | ignored |
| `stopCharging(DockOperationRequest)` | ignored |
| `rebootAsset(DockOperationRequest)` | ignored |
| `bootSubAsset(DockOperationRequest)` | `true`/`false` — boot the paired sub-asset on or off |
| `debugMode(DockOperationRequest)` | `true`/`false` — enable/disable debug mode |
| `changeAcMode(DockOperationRequest)` | ignored |
| `takePhoto(DockOperationRequest)` | ignored |
| `liveStreamSplitScreen(LiveStreamSplitScreenRequest)` | `enabled` (`true`/`false`) — toggle split-screen live view |

## Capabilities & custom commands

| Method | Returns | Purpose |
| --- | --- | --- |
| `getCapabilities(String sn)` | `CapabilitySnapshot` | Live capability snapshot — every command this asset currently supports, including vendor-defined ones with no built-in method |
| `sendCustomCommand(CustomCommandRequest)` | `CustomCommandResponse` | Invoke a command by id, e.g. from a `CapabilityDescriptor` via the `forCapability` factory |

`CapabilitySnapshot.snapshotState` is e.g. `CAPABILITY_SNAPSHOT_STATE_CURRENT` or `..._STALE` (stale
when the asset is unreachable). Each `CapabilityDescriptor` carries: `commandId`, `displayName`,
`description`, `state`, `unavailableReason` (set when `state` isn't available), `targetType`
(`ASSET`/`SUB_ASSET`/`PAYLOAD`/`COMPONENT`), `targetRef`, `schemaVersion`, and
`metadata`/`constraints`/`inputSchema`/`outputSchema` maps describing the command's parameters.
