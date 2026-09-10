# Edge SDK -- Live Data Service

The `LiveDataService` interface manages persistent gRPC streams between the edge adapter and the platform's Live Data Service, for three kinds of outbound data: **telemetry**, **detections**, and **notifications**. It provides both a POJO-based API (recommended for most use cases) and a raw Proto-based API for advanced scenarios.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Telemetry](#telemetry)
- [Detections](#detections)
- [Notifications](#notifications)
- [Stream Management](#stream-management)
- [Configuration](#configuration)
- [Best Practices](#best-practices)

---

## Overview

Edge adapters continuously push data to the Live Data Service, where it's broadcast to Client SDK consumers, shown live in the Admin Console, and stored for historical analysis:

- **Telemetry** -- position, battery, environmental readings, camera state, and more.
- **Detections** -- AI/vision detection results (e.g. from a `DetectTask`-style Skill).
- **Notifications** -- asset online/offline events, and progress/completion events for commands accepted asynchronously via `CommandResult.accepted(...)` (see [Edge Adapter](edge-sdk-adapter.md#task-execution)).

The `LiveDataService` abstracts the complexity of managing gRPC streams: one persistent stream per device per data kind, with automatic reconnection on failure (exponential backoff, 1s to 30s, 20% jitter, up to 10 attempts).

---

## How It Works

```
Your Adapter Code
      |
      v
LiveDataService.produce*(...)
      |
      v
Mapper (POJO --> Proto)
      |
      v
Per-device stream --> Live Data Service (platform)
```

- **One stream per device, per data kind.** Reused across subsequent pushes.
- **Automatic reconnection** with exponential backoff; a final failed attempt schedules a retry after 30 seconds.
- **Graceful shutdown** via `@PreDestroy` -- all streams are closed on application shutdown.
- **Thread-safe** -- device-to-stream mappings are stored in a `ConcurrentHashMap`.

---

## Telemetry

### Build the telemetry payload

`TelemetryData` is a single class with two nested detail types, `AssetDetails` and `SubAssetDetails` -- set exactly one of `.asset(...)` / `.subAsset(...)` depending on whether the reading describes the registered top-level **Asset** itself or one of its child **SubAssets**. Shared position fields (`latitude`, `longitude`, `absoluteAltitude`, `relativeAltitude`, `windSpeed`, `heading`) live directly on `TelemetryData`, not duplicated per source.

This is a source/context distinction, not a device-category one. An Asset can be a drone, a dock, a ground vehicle, a sensor gateway or a camera. If your adapter registers a drone as a standalone Asset with no parent, its telemetry uses `.asset(...)` and `getSourceType()` correctly returns `ASSET`. If the drone belongs to a dock, the dock is the Asset and the drone's readings use `.subAsset(...)`. Both configurations are supported -- see [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md).

```java
import com.zqnt.utils.edge.sdk.domains.TelemetryData;
import com.zqnt.sdk.edge.adapter.domains.TelemetryRequestData;
import java.time.LocalDateTime;
import java.util.UUID;

TelemetryData.AssetDetails assetDetails = TelemetryData.AssetDetails.builder()
    .environmentTemp(22.5f)
    .humidity(65.0f)
    .build();

TelemetryData telemetry = TelemetryData.builder()
    .id(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .sn("YOUR_DEVICE_SN")
    .latitude(47.3769)
    .longitude(8.5417)
    .absoluteAltitude(450.0f)
    .asset(assetDetails)
    .build();

TelemetryRequestData data = TelemetryRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .tid(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .telemetry(telemetry)
    .build();
```

### Send it

```java
import com.zqnt.sdk.edge.livedata.application.LiveDataService;

private final LiveDataService liveDataService;

public void sendTelemetry(TelemetryRequestData data) {
    liveDataService.produceTelemetryData(data)
        .thenRun(() -> log.debug("Telemetry sent for {}", data.getSn()))
        .exceptionally(err -> {
            log.error("Error sending telemetry", err);
            return null;
        });
}
```

### Sub-asset (drone/vehicle) example

```java
TelemetryData.SubAssetDetails subAssetDetails = TelemetryData.SubAssetDetails.builder()
    .horizontalSpeed(5.2f)
    .verticalSpeed(0.0f)
    .mode(SubAssetMode.SUBASSET_MODE_MANUAL)
    .batteryInformation(TelemetryData.BatteryInformation.builder()
        .percentage("87")
        .build())
    .build();

TelemetryData telemetry = TelemetryData.builder()
    .id(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .sn("YOUR_DRONE_SN")
    .latitude(47.3769)
    .longitude(8.5417)
    .absoluteAltitude(120.0f)
    .subAsset(subAssetDetails)
    .build();
```

See [Models Reference](edge-sdk-models.md#telemetrydata) for the full field list.

### Proto-based API (advanced)

```java
import com.zqnt.utils.livedata.proto.ProduceTelemetryRequest;

ProduceTelemetryRequest protoRequest = ProduceTelemetryRequest.newBuilder()
    .setBase(RequestBase.newBuilder()
        .setSn("YOUR_DEVICE_SN")
        .setTid(UUID.randomUUID().toString())
        .setTimestamp(ProtobufHelpers.now())
        .build())
    // ... set telemetry fields
    .build();

liveDataService.produceTelemetry("YOUR_DEVICE_SN", protoRequest)
    .thenRun(() -> log.debug("Proto telemetry sent"))
    .exceptionally(err -> {
        log.error("Error", err);
        return null;
    });
```

Both APIs share the same underlying stream infrastructure, so there is no performance difference.

---

## Detections

Push AI/vision detection results the same way as telemetry:

```java
import com.zqnt.sdk.edge.adapter.domains.DetectionRequestData;
import com.zqnt.sdk.edge.adapter.domains.DetectionRequestData.DetectionResultData;
import com.zqnt.sdk.edge.adapter.domains.DetectionRequestData.BoundingBoxData;

DetectionRequestData batch = DetectionRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .tid(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .streamUrl("rtmp://...")
    .detections(List.of(
        DetectionResultData.builder()
            .objectType("person")
            .confidence(0.91f)
            .boundingBox(BoundingBoxData.builder().x(120f).y(80f).width(64f).height(128f).build())
            .build()
    ))
    .build();

liveDataService.produceDetectionData(batch)
    .thenRun(() -> log.debug("Detections sent"))
    .exceptionally(err -> {
        log.error("Error sending detections", err);
        return null;
    });
```

---

## Notifications

Notifications cover two cases: reporting an asset's online/offline transitions, and reporting progress/completion of a command your adapter accepted asynchronously (see [`CommandResult.accepted(...)`](edge-sdk-adapter.md#commandresult)). Exactly one event field should be set per call.

```java
import com.zqnt.sdk.edge.adapter.domains.NotificationRequestData;
import com.zqnt.sdk.edge.adapter.domains.NotificationRequestData.CommandExecutionEventData;
import com.zqnt.utils.events.proto.CommandExecutionStatus;

// Report progress for a command you previously accepted with an externalExecutionId
NotificationRequestData progress = NotificationRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .timestamp(LocalDateTime.now())
    .commandExecutionEvent(CommandExecutionEventData.builder()
        .externalExecutionId(executionId)
        .commandId("mission.waypoint.execute")
        .status(CommandExecutionStatus.COMMAND_EXECUTION_STATUS_RUNNING)
        .progress(0.42f)
        .assetSn("YOUR_DEVICE_SN")
        .build())
    .build();

liveDataService.produceNotificationData(progress)
    .exceptionally(err -> {
        log.error("Error sending notification", err);
        return null;
    });
```

```java
import com.zqnt.sdk.edge.adapter.domains.NotificationRequestData.AssetStatusEventData;

NotificationRequestData assetOffline = NotificationRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .timestamp(LocalDateTime.now())
    .assetStatusEvent(AssetStatusEventData.builder()
        .sn("YOUR_DEVICE_SN")
        .online(false)
        .message("Lost connection to device")
        .build())
    .build();

liveDataService.produceNotificationData(assetOffline);
```

---

## Stream Management

Each of telemetry, detections, and notifications has its own stream lifecycle:

```java
liveDataService.closeStream("YOUR_DEVICE_SN");           // telemetry stream for one device
liveDataService.closeDetectionStream("YOUR_DEVICE_SN");   // detection stream for one device
liveDataService.closeNotificationStream("YOUR_DEVICE_SN"); // notification stream for one device
liveDataService.closeAllStreams();                        // all telemetry streams (shutdown)
```

The `LiveDataServiceImpl` also registers a `@PreDestroy` callback that automatically closes streams with a 10-second timeout on application shutdown.

---

## Configuration

The Live Data Service gRPC client is configured in `application.properties`:

```properties
quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003
quarkus.grpc.clients.live-data-service.keep-alive-without-calls=true
```

For container deployments:

```properties
quarkus.grpc.clients.live-data-service.host=live-data-service
quarkus.grpc.clients.live-data-service.port=8003
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.

---

## Best Practices

1. **Send telemetry at a reasonable frequency.** Sending too fast can overwhelm the gRPC stream and the platform. For most assets, 1-5 Hz is appropriate. For OSD (on-screen display) data from drones, the typical rate is 2 Hz.

2. **Use the POJO API unless you have a specific reason not to.** The mapper handles all Proto conversions for you, including timestamp mapping between `LocalDateTime` and Protobuf `Timestamp`.

3. **Set exactly one of `asset`/`subAsset`** on `TelemetryData` -- `getSourceType()` (and platform-side routing) depends on it.

4. **Include a transaction ID.** Setting `tid` on every message enables end-to-end tracing across the system.

5. **Correlate async commands with `externalExecutionId`.** If you returned `CommandResult.accepted(...)` for a command, use the same id in subsequent `CommandExecutionEventData` notifications so the platform can track and later cancel that specific run.

6. **Do not manually manage streams.** Let the SDK handle stream creation, reconnection, and teardown. If you need to reset a stream, call the relevant `close*Stream(deviceSn)` and the next `produce*` call will create a new one automatically.

7. **Handle shutdown gracefully.** If your adapter has its own shutdown logic, call `closeAllStreams()` before tearing down other resources. The SDK does this automatically via `@PreDestroy`, but explicit ordering can prevent race conditions.
