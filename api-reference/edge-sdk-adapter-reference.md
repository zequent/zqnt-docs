# Edge SDK — Edge Adapter API Reference

> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Edge Adapter reference](edge-sdk-adapter-reference-1.3.md).

Method reference for `EdgeAdapterService`, the interface your adapter implements. For a narrative
introduction and worked examples, see the [Edge Adapter guide](../edge-sdk/edge-sdk-adapter.md).

Every method is a `default` method returning `NOT_IMPLEMENTED` — override only what your device
supports. Every method returns `CompletableFuture<CommandResult>`, except `getCapabilities`
(`CompletableFuture<CurrentCapabilities>`).

## How commands arrive

The platform runs work as Skills. Each command node of a run is dispatched to your adapter by its
command id: a built-in id lands on the matching method below, any other id on `sendCustomCommand`.

- **Waits for your report.** `flight.takeoff`, `navigation.go_to`, `gimbal.look_at` and
  `flight.return_to_home`, and a custom command that returns no result, are *accepted*: the node
  stays running until you report a command execution event for it (`SUCCEEDED`, `FAILED`, ...). Return
  `CommandResult.accepted(message, externalExecutionId, sn)` with your own execution id; without one,
  the platform uses the request's transaction id. See
  [Edge Adapter guide — Custom Commands](../edge-sdk/edge-sdk-adapter.md#custom-commands).
- **Done on success.** Every other built-in command is complete when you return success.
- **Cancel.** The platform cancels a running command with `cancelExecution(sn, externalExecutionId)`.

## Flight

| Method | Command id | Description |
| --- | --- | --- |
| `takeOff(TakeOffRequest)` | `flight.takeoff` | Take off and fly to the target coordinates |
| `goTo(GoToRequest)` | `navigation.go_to` | Fly to the target coordinates; altitude is relative to the takeoff point |
| `returnToHome(ReturnToHomeRequest)` | `flight.return_to_home` | Return home; `altitude` optional |
| `enterManualControl(String sn)` | `flight.manual.enter` | Enter manual (stick) control |
| `exitManualControl(String sn)` | `flight.manual.exit` | Leave manual control |
| `manualControlInput(ManualControlInput)` | — | Called once per stick-input frame while a manual control session is active |

## Camera, gimbal and stream

| Method | Command id | Description |
| --- | --- | --- |
| `lookAt(LookAtRequest)` | `gimbal.look_at` | Point the camera at coordinates (`locked`, `payloadIndex` optional) |
| `enableGimbalTracking(String sn, boolean enabled)` | `gimbal.tracking` | Gimbal tracking on/off |
| `takePhoto(TakePhotoRequest)` | `camera.take_photo` | Capture a still photo |
| `changeLens(ChangeLensRequest)` | `camera.change_lens` | Switch the active lens |
| `changeZoom(ChangeZoomRequest)` | `camera.change_zoom` | Set the zoom level for a lens |
| `startLiveStream(LiveStreamStartRequest)` | `stream.start` | Start publishing video to the stream server URL in the request |
| `stopLiveStream(LiveStreamStopRequest)` | `stream.stop` | Stop publishing |
| `liveStreamSplitScreen(String sn, boolean enabled)` | `stream.split_screen` | Split-screen view across lenses on/off |

## Dock and asset

| Method | Command id | Description |
| --- | --- | --- |
| `openCover(String sn)` | `dock.open_cover` | Open the dock cover |
| `closeCover(String sn, Boolean force)` | `dock.close_cover` | Close the dock cover, optionally forced |
| `startCharging(String sn)` | `dock.start_charging` | Start charging the drone |
| `stopCharging(String sn)` | `dock.stop_charging` | Stop charging |
| `rebootAsset(String sn)` | `asset.reboot` | Reboot the asset |
| `bootUpSubAsset(String sn)` / `bootDownSubAsset(String sn)` | `asset.boot_sub_asset` | Power the sub-asset (drone) on / off |
| `enterRemoteDebugMode(String sn)` / `closeRemoteDebugMode(String sn)` | `asset.remote_debug` | Remote debug mode on / off |
| `changeAcMode(String sn, String mode)` | `asset.change_ac_mode` | Set the air conditioner; `mode` is `AIR_CONDITIONER_IDLE`, `_COOL`, `_HEAT` or `_DEHUMIDIFICATION` |

## Custom commands

| Method | Description |
| --- | --- |
| `sendCustomCommand(String sn, String componentId, String commandType, Map<String, Object> params)` | Every command id without a built-in method — vendor-, payload- and mission-specific commands (e.g. `mission.waypoint.execute`). `componentId` is the target's `targetRef` (e.g. which payload), or `null` |

Advertise each one through `getCapabilities` so it can be used in Skills. See
[Command ID naming convention](../edge-sdk/edge-sdk-adapter.md#command-id-naming-convention).

## Capabilities and cancellation

| Method | Returns | Description |
| --- | --- | --- |
| `getCapabilities(String sn)` | `CurrentCapabilities` | What this asset supports right now — see [Capability reporting](#capability-reporting) |
| `cancelExecution(String sn, String externalExecutionId)` | `CommandResult` | Stop a running command you accepted earlier |

### Capability reporting

`getCapabilities` returns the device's live capability snapshot. The platform reads it to know which
command ids an asset offers (built-in ones too), shows it to operators and client applications, and
records each command in the Skill Registry. The default returns an empty snapshot. See
[Edge Adapter guide — Custom Commands](../edge-sdk/edge-sdk-adapter.md#custom-commands) and the
[models reference](edge-sdk-models.md) for the `Capability` fields.

## CommandResult

```java
CommandResult.success("Cover opened", sn);                 // done
CommandResult.success("Cover opened", tid, sn);            // done, with a transaction id
CommandResult.accepted("Mission started", executionId, sn); // running; report its outcome later
CommandResult.error("Cover is blocked", sn);               // failed
CommandResult.error("Cover is blocked", tid, sn);
CommandResult.notImplemented("Not supported", sn);         // what the defaults return
```

`CommandResult.CommandResultType`: `SUCCESS`, `ACCEPTED`, `ERROR`, `NOT_IMPLEMENTED`.

## Error handling

An exception thrown by your method is turned into an error response:

| Exception | Error code |
| --- | --- |
| `IllegalArgumentException`, `UnsupportedOperationException` | `ERROR_CODE_CLIENT` |
| `TimeoutException` and everything else | `ERROR_CODE_SYSTEM` |

A returned `CommandResult.error(...)` is answered with `ERROR_CODE_ASSET`, a `notImplemented(...)`
with `ERROR_CODE_CLIENT`. Prefer returning `error(...)` for expected failures.

## EdgeAdapterServiceImpl

If your adapter extends `EdgeAdapterServiceImpl` instead of implementing the interface directly, it
gets overloads without the `sn` parameter that use the configured serial number
(`EdgeClientConfig.getSn()`): `openCover()`, `closeCover()`, `startCharging()`, `stopCharging()`,
`rebootAsset()`, `bootUpSubAsset()`, `bootDownSubAsset()`, `enterManualControl()`,
`exitManualControl()`, `getCapabilities()`, `enableGimbalTracking(boolean)`, `changeAcMode(String)`.
