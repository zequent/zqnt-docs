# Zequent Client SDK — Remote Control

`client.remoteControl()` sends direct, imperative commands to a connected asset — flight ops, dock ops, manual control, and dynamic capability discovery for payload/integrator-defined commands. Every call targets a single asset by serial number (`sn`).

Full method-by-method reference: [Remote Control API Reference](../api-reference/client-sdk-remote-control.md).

For response semantics — what "success" actually means, and how command responses relate to progress/telemetry — see [Functional Responses](FUNCTIONAL_RESPONSES.md).

## Flight ops

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

`missionId`/`taskId` on `TakeoffRequest`/`GoToRequest` are optional — set them to correlate the command with a mission/task you already created via [Connector](CONNECTOR.md#missions-and-tasks-are-records-not-flights).

## Manual control

`startManualControlInput(sn, assetId)` returns a `ManualControlInputSession` — call `sendInput` repeatedly, then `complete()` to close the stream and get the final response. `enterManualControl`/`exitManualControl` take/release exclusive control around the session:

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

All of these take a `DockOperationRequest` (`sn`, `assetId`, optional `value`) except
`liveStreamSplitScreen` — what `value` means differs per method (e.g. forcing `closeCover`, toggling
`debugMode`); see the [reference](../api-reference/client-sdk-remote-control.md#dock--asset-ops) for
the full list.

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

## See also

- [Remote Control API Reference](../api-reference/client-sdk-remote-control.md) — every method, grouped by area
- [Functional Responses](FUNCTIONAL_RESPONSES.md) — what a response actually confirms
