# Edge SDK -- Edge Adapter Service

The `EdgeAdapterService` interface is the core contract of the Edge SDK. Every edge adapter must provide a CDI bean that implements this interface. The SDK ships with a default implementation (`EdgeAdapterServiceImpl`) whose methods all return `NOT_IMPLEMENTED`, so you only need to override the commands that your particular hardware supports.

## Table of Contents

- [How It Works](#how-it-works)
- [Creating an Adapter](#creating-an-adapter)
- [Command Reference](#command-reference)
  - [Flight Control](#flight-control)
  - [Dock Operations](#dock-operations)
  - [Camera and Gimbal](#camera-and-gimbal)
  - [Manual Control](#manual-control)
  - [Live Streaming](#live-streaming)
  - [Debug and Maintenance](#debug-and-maintenance)
  - [Task Execution](#task-execution)
  - [Capability Reporting](#capability-reporting)
- [CommandResult](#commandresult)
- [Default Implementation Convenience Methods](#default-implementation-convenience-methods)
- [gRPC Layer](#grpc-layer)
- [Error Handling](#error-handling)
- [Best Practices](#best-practices)

---

## How It Works

When your Quarkus application starts, the SDK registers a gRPC service (`EdgeAdapterGrpcServiceImpl`) that receives commands from the platform and delegates them to your `EdgeAdapterService` bean. The flow is:

```
Platform Services  --(gRPC)-->  EdgeAdapterGrpcServiceImpl  --(delegates)-->  Your EdgeAdapterService implementation
```

Every command method returns `CompletableFuture<CommandResult>`, which means your implementation can be fully asynchronous. The gRPC layer wraps it in a Mutiny `Uni` automatically.

---

## Creating an Adapter

### Step 1: Implement the Interface

Create a CDI bean that implements `EdgeAdapterService`. The `@ApplicationScoped` annotation ensures there is a single instance across the application lifecycle.

```java
package com.example.edge;

import com.zqnt.sdk.edge.adapter.application.EdgeAdapterService;
import com.zqnt.sdk.edge.adapter.domains.*;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.CompletableFuture;

@ApplicationScoped
public class MyDeviceAdapter implements EdgeAdapterService {

    @Override
    public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
        // Call your device SDK/API here
        boolean success = myDevice.initiateTakeoff(
            request.getCoordinates().getLatitude(),
            request.getCoordinates().getLongitude(),
            request.getCoordinates().getAltitude()
        );

        if (success) {
            return CompletableFuture.completedFuture(
                CommandResult.success("Takeoff initiated", request.getTid(), request.getSn())
            );
        } else {
            return CompletableFuture.completedFuture(
                CommandResult.error("Takeoff failed: device busy", request.getSn())
            );
        }
    }
}
```

### Step 2: Asynchronous Implementation

If your device SDK provides asynchronous or callback-based APIs, use `CompletableFuture` accordingly:

```java
@Override
public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
    CompletableFuture<CommandResult> future = new CompletableFuture<>();

    myDevice.takeoffAsync(request.getCoordinates(), new DeviceCallback() {
        @Override
        public void onSuccess() {
            future.complete(CommandResult.success("Takeoff complete", request.getSn()));
        }

        @Override
        public void onError(String errorMsg) {
            future.complete(CommandResult.error(errorMsg, request.getSn()));
        }
    });

    return future;
}
```

### Step 3: Override Only What You Support

Any method that you do not override will automatically return a `NOT_IMPLEMENTED` result to the caller. This is by design -- a dock adapter may support `openCover` and `startCharging` but not `takeOff` (which is a drone-level command), and that is perfectly fine.

---

## Command Reference

### Flight Control

| Method | Parameters | Description |
|--------|-----------|-------------|
| `takeOff(TakeOffRequest)` | sn, tid, coordinates | Initiate takeoff at the given coordinates |
| `returnToHome(ReturnToHomeRequest)` | sn, tid, altitude | Return the sub-asset to its home position |
| `goTo(GoToRequest)` | sn, tid, coordinates | Navigate to the specified coordinates |

Example:

```java
@Override
public CompletableFuture<CommandResult> goTo(GoToRequest request) {
    deviceApi.flyTo(
        request.getCoordinates().getLatitude(),
        request.getCoordinates().getLongitude(),
        request.getCoordinates().getAltitude()
    );
    return CompletableFuture.completedFuture(
        CommandResult.success("Navigation started", request.getTid(), request.getSn())
    );
}
```

### Dock Operations

| Method | Parameters | Description |
|--------|-----------|-------------|
| `openCover(String sn)` | sn | Open the dock cover |
| `closeCover(String sn, Boolean force)` | sn, force | Close the dock cover, optionally forcing it |
| `startCharging(String sn)` | sn | Start charging the sub-asset |
| `stopCharging(String sn)` | sn | Stop charging the sub-asset |
| `rebootAsset(String sn)` | sn | Reboot the asset (dock) |
| `bootUpSubAsset(String sn)` | sn | Power on the sub-asset (drone) |
| `bootDownSubAsset(String sn)` | sn | Power off the sub-asset (drone) |

Example:

```java
@Override
public CompletableFuture<CommandResult> openCover(String sn) {
    dockApi.sendCommand("open_cover");
    return CompletableFuture.completedFuture(
        CommandResult.success("Cover opening", sn)
    );
}
```

### Camera and Gimbal

| Method | Parameters | Description |
|--------|-----------|-------------|
| `lookAt(LookAtRequest)` | sn, lat, lon, alt, locked, payloadIndex | Point the camera at coordinates |
| `changeLens(ChangeLensRequest)` | sn, lens, videoId | Switch the active camera lens |
| `changeZoom(ChangeZoomRequest)` | sn, lens, payloadIndex, zoom | Adjust the camera zoom level |
| `takePhoto(TakePhotoRequest)` | sn, payloadIndex | Capture a still photo |
| `enableGimbalTracking(String sn, boolean enabled)` | sn, enabled | Enable or disable gimbal tracking mode |
| `liveStreamSplitScreen(String sn, boolean enabled)` | sn, enabled | Toggle split-screen view across multiple lenses/payloads |

Example:

```java
@Override
public CompletableFuture<CommandResult> lookAt(LookAtRequest request) {
    cameraApi.pointAt(
        request.getLatitude(),
        request.getLongitude(),
        request.getAltitude(),
        request.getLocked()
    );
    return CompletableFuture.completedFuture(
        CommandResult.success("Camera pointing", request.getSn())
    );
}
```

### Manual Control

| Method | Parameters | Description |
|--------|-----------|-------------|
| `enterManualControl(String sn)` | sn | Enter manual (joystick) control mode |
| `exitManualControl(String sn)` | sn | Exit manual control mode |
| `manualControlInput(ManualControlInput)` | input | Called once per incoming stick-input frame while a manual control session is active |

The gRPC layer calls `manualControlInput` once for every frame it receives from the client's manual-control stream — each call carries one `ManualControlInput` with roll, pitch, yaw, throttle, and gimbalPitch values. You don't manage the stream yourself; just forward each frame to your device.

Example:

```java
@Override
public CompletableFuture<CommandResult> enterManualControl(String sn) {
    deviceApi.enableDrcMode(sn);
    return CompletableFuture.completedFuture(
        CommandResult.success("Manual control mode entered", sn)
    );
}

@Override
public CompletableFuture<CommandResult> manualControlInput(ManualControlInput input) {
    deviceApi.sendStickCommand(
        input.getRoll(), input.getPitch(),
        input.getYaw(), input.getThrottle()
    );
    return CompletableFuture.completedFuture(
        CommandResult.success("Stick input applied", input.getSn())
    );
}
```

### Live Streaming

| Method | Parameters | Description |
|--------|-----------|-------------|
| `startLiveStream(LiveStreamStartRequest)` | sn, tid, videoId, streamServer, videoType | Start a video live stream |
| `stopLiveStream(LiveStreamStopRequest)` | sn, tid, videoId | Stop a video live stream |

Example:

```java
@Override
public CompletableFuture<CommandResult> startLiveStream(LiveStreamStartRequest request) {
    deviceApi.startStream(request.getVideoId(), request.getStreamServer());
    return CompletableFuture.completedFuture(
        CommandResult.success("Live stream started", request.getSn())
    );
}
```

### Debug and Maintenance

| Method | Parameters | Description |
|--------|-----------|-------------|
| `enterRemoteDebugMode(String sn)` | sn | Enter remote debug mode on the device |
| `closeRemoteDebugMode(String sn)` | sn | Exit remote debug mode |
| `changeAcMode(String sn, String mode)` | sn, mode | Change the air conditioner mode of the asset |

### Task Execution

| Method | Parameters | Description |
|--------|-----------|-------------|
| `prepareTask(String taskId, String tid)` | taskId, tid | Prepare a task for execution. Receives only a task ID — see the note below |
| `startTask(String taskId, String tid)` | taskId, tid | Start executing a previously prepared task. Receives only a task ID — see the note below |
| `pauseTask(String taskId)` | taskId | Pause a running task |
| `resumeTask(String taskId)` | taskId | Resume a paused task |
| `stopTask(String taskId)` | taskId | Stop a running task |
| `cancelExecution(String sn, String externalExecutionId)` | sn, externalExecutionId | Cancel an asynchronously-running command previously accepted via `sendCustomCommand` |

> **The task methods receive only a task ID.** To act on one, resolve it with
> `ConnectorService.getTaskById(taskId)` and read the `WaypointTaskConfig` off the returned
> `TaskDTO` — that is what the DJI adapter does to build and upload its KMZ. SAPIENT implements
> them too, because its own protocol owns the task that ID refers to.
>
> The alternative is to skip them entirely and accept `mission.waypoint.execute` through
> `sendCustomCommand` (below), where the waypoints and configuration arrive inline and no lookup is
> needed — that is what the MAVLink adapter and the simulator do. Both approaches are valid;
> leaving the task methods unimplemented returns `NOT_IMPLEMENTED`, which callers handle. Whichever
> you choose, document it, because the two are not interchangeable from a customer application's
> point of view.

### Custom Commands

| Method | Parameters | Description |
|--------|-----------|-------------|
| `sendCustomCommand(String sn, String componentId, String commandType, Map<String, Object> params)` | sn, componentId, commandType, params | Handle a command that doesn't map to a standard method above. Set `externalExecutionId` on the result if the command keeps running asynchronously, so `cancelExecution`/`stopTask` can reference it later. |

Example:

```java
@Override
public CompletableFuture<CommandResult> sendCustomCommand(String sn, String componentId,
        String commandType, Map<String, Object> params) {
    if ("mission.waypoint.execute".equals(commandType)) {
        String executionId = deviceApi.startWaypointMission(params);
        return CompletableFuture.completedFuture(
            CommandResult.accepted("Waypoint mission started", executionId, sn)
        );
    }
    return CompletableFuture.completedFuture(CommandResult.notImplemented("Unknown command", sn));
}
```

#### Command ID naming convention

Every built-in command above maps to a well-known, vendor-neutral `command_id` string. Custom commands should follow the same convention:

- **Vendor-neutral** — never encode a vendor name (`dji.takeoff` is wrong); the same id should be implementable by any adapter.
- **`domain.action`** for a single atomic command — a domain (`flight`, `navigation`, `dock`, `asset`, `camera`, `gimbal`, `stream`) and a snake_case action (`return_to_home`, `go_to`, `change_lens`).
- **`domain.subtype.action`** only when a domain genuinely has distinct execution variants, e.g. `mission.waypoint.execute`.
- **`custom.` prefix** for tenant/user-defined commands, so they never collide with a future built-in of the same name.
- Commands under `flight.`, `navigation.`, `dock.`, `mission.`, and `asset.reboot` are treated as high-risk by the platform's execution safety checks (e.g. may require human approval) — keep new movement-capable commands under one of those prefixes rather than introducing an unrecognized domain.

Example:

```java
@Override
public CompletableFuture<CommandResult> prepareTask(String taskId, String tid) {
    // Fetch waypoints, generate flight plan, upload to device
    missionService.generateAndUpload(taskId);
    return CompletableFuture.completedFuture(
        CommandResult.success("Task prepared", tid, taskId)
    );
}

@Override
public CompletableFuture<CommandResult> startTask(String taskId, String tid) {
    deviceApi.executeFlightPlan(taskId);
    return CompletableFuture.completedFuture(
        CommandResult.success("Task started", tid, taskId)
    );
}
```

### Capability Reporting

| Method | Parameters | Description |
|--------|-----------|-------------|
| `getCapabilities(String sn)` | sn | Return the set of capabilities this adapter supports |

Override this method to tell the platform exactly which commands your adapter supports and any metadata about them:

```java
@Override
public CompletableFuture<CurrentCapabilities> getCapabilities(String sn) {
    Set<Capability> capabilities = Set.of(
        new Capability("takeOff", "Initiate drone takeoff", true, null, Map.of()),
        new Capability("openCover", "Open dock cover", true, null, Map.of()),
        new Capability("goTo", "Navigate to coordinates", true, null, Map.of()),
        new Capability("manualControlInput", "Joystick control", false,
            "DRC mode not available", Map.of())
    );

    return CompletableFuture.completedFuture(
        CurrentCapabilities.of(sn, AssetTypeEnum.ASSET_TYPE_DOCK, capabilities)
    );
}
```

---

## CommandResult

Every adapter command returns a `CommandResult`. Use the static factory methods to create results:

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
// stopTask/cancelExecution call can reference this specific run
CommandResult.accepted("Waypoint mission started", externalExecutionId, sn);

// Not Implemented (used by default methods)
CommandResult.notImplemented("Command not supported", sn);
```

The `CommandResult.ResultType` enum has these values:
- `SUCCESS` -- command executed successfully
- `ERROR` -- command failed
- `ACCEPTED` -- command was accepted and is still running asynchronously (use `isAccepted()` to check)
- `NOT_IMPLEMENTED` -- command is not supported by this adapter

---

## Default Implementation Convenience Methods

The provided `EdgeAdapterServiceImpl` class extends the interface with convenience methods that automatically use the configured serial number from `EdgeClientConfig.sn()`. These include:

- `openCover()` / `closeCover()` (no sn parameter)
- `startCharging()` / `stopCharging()`
- `rebootAsset()`
- `bootUpSubAsset()` / `bootDownSubAsset()`
- `enterManualControl()` / `exitManualControl()`
- `getCapabilities()`
- `enableGimbalTracking(boolean)`
- `changeAcMode(String mode)`

If your adapter extends `EdgeAdapterServiceImpl` instead of implementing `EdgeAdapterService` directly, these methods are available automatically.

---

## gRPC Layer

The `EdgeAdapterGrpcServiceImpl` class is registered as a `@GrpcService`. It:

1. Receives incoming gRPC requests from the platform (Remote Control Service, Client SDK).
2. Maps Proto request messages to SDK model POJOs using `ProtoJsonMapper`.
3. Delegates to your `EdgeAdapterService` implementation.
4. Converts `CommandResult` back to an `EdgeResponse` Proto message.
5. Handles errors with proper gRPC error codes and `GlobalErrorMessage`.

You typically do not need to interact with this class directly. It is wired automatically by the CDI container and the Quarkus gRPC extension.

---

## Error Handling

Exceptions thrown by your adapter code are caught by the gRPC layer and mapped to error responses:

| Exception Type | gRPC Error Code |
|----------------|-----------------|
| `IllegalArgumentException` | `CLIENT_ERROR` |
| `UnsupportedOperationException` | `CLIENT_ERROR` |
| `TimeoutException` | `SYSTEM_ERROR` |
| All other exceptions | `SYSTEM_ERROR` |

You can also return explicit error results using `CommandResult.error(...)` instead of throwing exceptions for expected failure conditions.

---

## Best Practices

1. **Override selectively.** Only implement the commands your hardware actually supports. The default `NOT_IMPLEMENTED` response gives callers a clear signal that a given command is not available.

2. **Use CompletableFuture properly.** Do not block inside your adapter methods. If your device SDK is blocking, wrap the call in `CompletableFuture.supplyAsync(...)`.

3. **Report capabilities.** Override `getCapabilities` to give the platform and its users an accurate view of what your device can do at any given moment.

4. **Use transaction IDs.** Pass the `tid` from the request through to the `CommandResult` so that command execution can be correlated across the system.

5. **Handle timeouts.** If your device takes time to respond, use `CompletableFuture.orTimeout(...)` or custom timeout logic so that callers are not left waiting indefinitely.

6. **Log meaningfully.** The gRPC layer already logs incoming commands. Focus your adapter logs on device-level events, errors, and state changes.
