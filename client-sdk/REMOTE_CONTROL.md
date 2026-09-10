# Zequent Client SDK — Remote Control

`client.remoteControl()` sends direct, imperative commands to a connected asset — flight ops, dock ops, manual control, and dynamic capability discovery for payload/integrator-defined commands. Every call targets a single asset by serial number (`sn`).

For response semantics — what "success" actually means, and how command responses relate to progress/telemetry — see [Functional Responses](FUNCTIONAL_RESPONSES.md).

## Flight ops

| Method | Purpose |
| --- | --- |
| `takeoff(TakeoffRequest)` | Take off and fly to a target lat/lon/altitude |
| `goTo(GoToRequest)` | Fly to a target lat/lon/altitude |
| `returnToHome(ReturnToHomeRequest)` | Return to home, optionally at a given altitude |
| `lookAt(LookAtRequest)` | Point the gimbal/camera at a lat/lon/altitude |

```java
var request = TakeoffRequest.builder()
    .sn("YOUR_DEVICE_SN")
    .latitude(52.520008f)
    .longitude(13.404954f)
    .altitude(50.0f)
    .build();

client.remoteControl().takeoff(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            log.warn("Takeoff failed: {}", response.getError().getErrorMessage());
            return;
        }
        System.out.println("Takeoff accepted: " + response.getTid());
    });
```

`missionId`/`taskId` on `TakeoffRequest`/`GoToRequest` are optional — set them to correlate the command with a mission/task you already created via [Connector](CONNECTOR.md#missions).

## Manual control

| Method | Purpose |
| --- | --- |
| `enterManualControl(ManualControlRequest)` | Take exclusive manual control of an asset |
| `exitManualControl(ManualControlRequest)` | Release manual control |
| `startManualControlInput(sn, assetId)` | Open a gRPC streaming session for continuous stick input |

`startManualControlInput` returns a `ManualControlInputSession` — call `sendInput` repeatedly, then `complete()` to close the stream and get the final response:

```java
try (ManualControlInputSession session = client.remoteControl().startManualControlInput(sn, assetId)) {
    ManualControlInput input = new ManualControlInput();
    input.setRoll(0.1f);
    input.setPitch(0.0f);
    input.setYaw(0.0f);
    input.setThrottle(0.5f);
    session.sendInput(input);

    RemoteControlResponse response = session.complete();
} catch (Exception e) {
    log.error("Manual control session failed", e);
}
```

## Dock & asset ops

All of these take a `DockOperationRequest` (`sn`, `assetId`, optional `value`) except `liveStreamSplitScreen`.

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

```java
var request = DockOperationRequest.builder()
    .sn("YOUR_DOCK_SN")
    .value(true)
    .build();

client.remoteControl().debugMode(request)
    .thenAccept(response -> System.out.println("Debug mode: " + response.isSuccess()));
```

## Capabilities & custom commands

Every connected asset can report a **live capability snapshot** — the set of commands it actually supports right now, including ones the platform has no built-in method for (e.g. a specific payload's vendor-defined commands). A capability snapshot is per-asset, live, and can go stale if the asset is unreachable.

```java
client.remoteControl().getCapabilities("YOUR_DEVICE_SN")
    .thenAccept(snapshot -> {
        System.out.println("Snapshot state: " + snapshot.getSnapshotState()); // e.g. "CAPABILITY_SNAPSHOT_STATE_CURRENT" or "..._STALE"
        snapshot.getCapabilities().forEach(cap ->
            System.out.println(cap.getCommandId() + " -> " + cap.getState()));
    });
```

Each `CapabilityDescriptor` has: `commandId`, `displayName`, `description`, `state`, `unavailableReason` (set when `state` isn't available), `targetType` (`ASSET`/`SUB_ASSET`/`PAYLOAD`/`COMPONENT`), `targetRef`, `schemaVersion`, and `metadata`/`constraints`/`inputSchema`/`outputSchema` maps describing the command's parameters.

Send one with `sendCustomCommand` — build the request from a `CapabilityDescriptor` you already have via the `forCapability` convenience factory, rather than filling in `commandType`/`componentId`/`targetType` by hand:

```java
CapabilityDescriptor searchlight = snapshot.getCapabilities().stream()
    .filter(cap -> cap.getCommandId().equals("searchlight.mode.set"))
    .findFirst()
    .orElseThrow();

var request = CustomCommandRequest.forCapability(
    "YOUR_DEVICE_SN", searchlight, Map.of("mode", "strobe"));

client.remoteControl().sendCustomCommand(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            log.warn("Custom command failed: {}", response.getError().getErrorMessage());
            return;
        }
        System.out.println("Result: " + response.getResult());
    });
```

## Error handling

`RemoteControlResponse`/`TakeoffResponse`/`CustomCommandResponse` all use `isSuccess()` + `getError().getErrorMessage()` rather than throwing for expected business errors. Handle both:

```java
client.remoteControl().goTo(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            log.warn("GoTo failed: {}", response.getError().getErrorMessage());
            return;
        }
        // command accepted
    })
    .exceptionally(err -> {
        log.error("Remote control call failed", err);
        return null;
    });
```
