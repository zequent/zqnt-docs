# Edge SDK — Edge Adapter API Reference

Exhaustive method reference for `EdgeAdapterService`. For a narrative introduction, the interface's
role in the gRPC flow, and worked examples, see the [Edge Adapter guide](../edge-sdk/edge-sdk-adapter.md).

Every method returns `CompletableFuture<CommandResult>` (`getCapabilities` returns
`CompletableFuture<CurrentCapabilities>`). All are `default` methods returning `NOT_IMPLEMENTED` —
override only what your adapter supports.

## Flight Control

| Method | Parameters | Description |
|--------|-----------|-------------|
| `takeOff(TakeOffRequest)` | sn, tid, coordinates | Initiate takeoff at the given coordinates |
| `returnToHome(ReturnToHomeRequest)` | sn, tid, altitude | Return the sub-asset to its home position |
| `goTo(GoToRequest)` | sn, tid, coordinates | Navigate to the specified coordinates |

## Dock Operations

| Method | Parameters | Description |
|--------|-----------|-------------|
| `openCover(String sn)` | sn | Open the dock cover |
| `closeCover(String sn, Boolean force)` | sn, force | Close the dock cover, optionally forcing it |
| `startCharging(String sn)` | sn | Start charging the sub-asset |
| `stopCharging(String sn)` | sn | Stop charging the sub-asset |
| `rebootAsset(String sn)` | sn | Reboot the asset (dock) |
| `bootUpSubAsset(String sn)` | sn | Power on the sub-asset (drone) |
| `bootDownSubAsset(String sn)` | sn | Power off the sub-asset (drone) |

## Camera and Gimbal

| Method | Parameters | Description |
|--------|-----------|-------------|
| `lookAt(LookAtRequest)` | sn, lat, lon, alt, locked, payloadIndex | Point the camera at coordinates |
| `changeLens(ChangeLensRequest)` | sn, lens, videoId | Switch the active camera lens |
| `changeZoom(ChangeZoomRequest)` | sn, lens, payloadIndex, zoom | Adjust the camera zoom level |
| `takePhoto(TakePhotoRequest)` | sn, payloadIndex | Capture a still photo |
| `enableGimbalTracking(String sn, boolean enabled)` | sn, enabled | Enable or disable gimbal tracking mode |
| `liveStreamSplitScreen(String sn, boolean enabled)` | sn, enabled | Toggle split-screen view across multiple lenses/payloads |

## Manual Control

| Method | Parameters | Description |
|--------|-----------|-------------|
| `enterManualControl(String sn)` | sn | Enter manual (joystick) control mode |
| `exitManualControl(String sn)` | sn | Exit manual control mode |
| `manualControlInput(ManualControlInput)` | input | Called once per incoming stick-input frame while a manual control session is active |

## Live Streaming

| Method | Parameters | Description |
|--------|-----------|-------------|
| `startLiveStream(LiveStreamStartRequest)` | sn, tid, videoId, streamServer, videoType | Start a video live stream |
| `stopLiveStream(LiveStreamStopRequest)` | sn, tid, videoId | Stop a video live stream |

## Debug and Maintenance

| Method | Parameters | Description |
|--------|-----------|-------------|
| `enterRemoteDebugMode(String sn)` | sn | Enter remote debug mode on the device |
| `closeRemoteDebugMode(String sn)` | sn | Exit remote debug mode |
| `changeAcMode(String sn, String mode)` | sn, mode | Change the air conditioner mode of the asset |

## Task Execution

| Method | Parameters | Description |
|--------|-----------|-------------|
| `prepareTask(String taskId, String tid)` | taskId, tid | Prepare a task for execution. Receives only a task ID — see the note below |
| `startTask(String taskId, String tid)` | taskId, tid | Start executing a previously prepared task. Receives only a task ID — see the note below |
| `pauseTask(String taskId)` | taskId | Pause a running task |
| `resumeTask(String taskId)` | taskId | Resume a paused task |
| `stopTask(String taskId)` | taskId | Stop a running task |

> **The task methods receive only a task ID.** To act on one, resolve it with
> `ConnectorService.getTaskById(taskId)` and read the `WaypointTaskConfig` off the returned
> `TaskDTO` — that is what the DJI adapter does to build and upload its KMZ. SAPIENT implements
> them too, because its own protocol owns the task that ID refers to. MAVLink and the simulator
> implement none of them — see [Custom Commands](../edge-sdk/edge-sdk-adapter.md#custom-commands)
> for the alternative path they use instead.

## Custom Commands

| Method | Parameters | Description |
|--------|-----------|-------------|
| `sendCustomCommand(String sn, String componentId, String commandType, Map<String, Object> params)` | sn, componentId, commandType, params | Handle a command that doesn't map to a standard method above |

See [Command ID naming convention](../edge-sdk/edge-sdk-adapter.md#command-id-naming-convention) for
how to name a custom command.

## Capability Reporting

| Method | Parameters | Description |
|--------|-----------|-------------|
| `getCapabilities(String sn)` | sn | Return the set of capabilities this adapter supports |

## CommandResult

Static factory methods on `CommandResult`:

```java
// Success without transaction ID
CommandResult.success("Message", sn);

// Success with transaction ID
CommandResult.success("Message", tid, sn);

// Error without transaction ID
CommandResult.error("Error description", sn);

// Error with transaction ID
CommandResult.error("Error description", tid, sn);

// Accepted, but still running asynchronously — pass externalExecutionId so a later
// stopTask call can reference this specific run
CommandResult.success("Waypoint mission started", vendorExecutionId, sn);

// Not Implemented (used by default methods)
CommandResult.notImplemented("Command not supported", sn);
```

`CommandResult.ResultType`:

| Value | Meaning |
|---|---|
| `SUCCESS` | command executed successfully |
| `ERROR` | command failed |
| `NOT_IMPLEMENTED` | command is not supported by this adapter |

## Default Implementation Convenience Methods

`EdgeAdapterServiceImpl` extends the interface with convenience overloads that automatically use the
configured serial number from `EdgeClientConfig.sn()`:

- `openCover()` / `closeCover()` (no sn parameter)
- `startCharging()` / `stopCharging()`
- `rebootAsset()`
- `bootUpSubAsset()` / `bootDownSubAsset()`
- `enterManualControl()` / `exitManualControl()`
- `getCapabilities()`
- `enableGimbalTracking(boolean)`
- `changeAcMode(String mode)`

Available automatically if your adapter extends `EdgeAdapterServiceImpl` instead of implementing
`EdgeAdapterService` directly.

## Error Handling

Exceptions thrown by your adapter code are caught by the gRPC layer and mapped to error responses:

| Exception Type | gRPC Error Code |
|----------------|-----------------|
| `IllegalArgumentException` | `CLIENT_ERROR` |
| `UnsupportedOperationException` | `CLIENT_ERROR` |
| `TimeoutException` | `SYSTEM_ERROR` |
| All other exceptions | `SYSTEM_ERROR` |

You can also return explicit error results using `CommandResult.error(...)` instead of throwing
exceptions for expected failure conditions.
