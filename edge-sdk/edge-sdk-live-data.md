# Edge SDK -- Live Data Service

The `LiveDataService` interface manages persistent gRPC streams between the edge adapter and the platform's Live Data Service, for three kinds of outbound data: **telemetry**, **detections**, and **notifications**. It provides both a POJO-based API (recommended for most use cases) and a raw Proto-based API for advanced scenarios.

Full method-by-method reference: [Live Data API Reference](../api-reference/edge-sdk-live-data-reference.md).

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Telemetry](#telemetry)
- [Detections](#detections)
- [Notifications](#notifications)
- [Stream Management](#stream-management)
- [Wiring and shutdown](#wiring-and-shutdown)
- [Configuration](#configuration)
- [Best Practices](#best-practices)

---

## Overview

Edge adapters continuously push data to the Live Data Service, where it's broadcast to Client SDK consumers, shown live in the Admin Console, and stored for historical analysis:

- **Telemetry** -- position, battery, environmental readings, camera state, and more.
- **Detections** -- AI/vision detection results.
- **Notifications** -- asset online/offline events, task progress/completion events, and (less commonly) mission-level events (see [API Reference — Notifications](../api-reference/edge-sdk-live-data-reference.md#notifications)).

The `LiveDataService` abstracts the complexity of managing gRPC streams: one persistent stream per device per data kind, with automatic reconnection on failure.

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
- **Automatic reconnection** with capped exponential backoff and no attempt limit — see
  [API Reference — Reconnection behavior](../api-reference/edge-sdk-live-data-reference.md#reconnection-behavior)
  for the exact numbers.
- **Thread-safe** -- device-to-stream mappings are stored in a `ConcurrentHashMap`.
- **Shutdown is not automatic.** The SDK registers no shutdown hook of its own — your adapter
  project has to call it, the same way it has to produce the `LiveDataService` bean in the first
  place. See [Wiring and shutdown](#wiring-and-shutdown).

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

See [Models Reference](../api-reference/edge-sdk-models.md#telemetrydata) for the full field list.

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
Detections and notifications have the same two-API shape — see the
[reference](../api-reference/edge-sdk-live-data-reference.md) for their Proto-based method signatures.

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

Notifications cover three cases: reporting an asset's online/offline transitions, reporting progress
or completion of a task your adapter is running, and (less commonly — no confirmed usage in any
current adapter) mission-level events. Exactly one event field should be set per call — see the
[reference](../api-reference/edge-sdk-live-data-reference.md#notifications) for the full field list of each.

```java
import com.zqnt.sdk.edge.adapter.domains.NotificationRequestData;
import com.zqnt.sdk.edge.adapter.domains.NotificationRequestData.TaskEventData;
import com.zqnt.utils.mission.proto.TaskStatus;
import com.zqnt.utils.mission.proto.TaskTypeProto;

// Report progress for a task your adapter is running
NotificationRequestData progress = NotificationRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .timestamp(LocalDateTime.now())
    .eventType(NotificationEventType.NOTIFICATION_EVENT_TASK)
    .taskEvent(TaskEventData.builder()
        .taskId(taskId)
        .taskType(TaskTypeProto.TASK_TYPE_WAYPOINT)
        .status(TaskStatus.TASK_RUNNING)
        .progress(0.42f)
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
liveDataService.closeAllStreams();                        // all telemetry streams — not detection/notification, see below
```

`closeAllStreams()` only closes telemetry streams despite the name — it does not touch detection or
notification streams. For a complete shutdown across all three, use `LiveDataServiceImpl.shutdown()`
instead (see below).

---

## Wiring and shutdown

The SDK does not wire `LiveDataService` into CDI by itself, and does not close anything on
application shutdown automatically — there is no `@PreDestroy`/`@Shutdown` hook inside the SDK.
Your adapter project has to produce the bean and shut it down itself, the same way the DJI adapter
does it:

```java
@ApplicationScoped
public class LiveDataProducer {

    private final TelemetryMapper telemetryMapper;
    private final DetectionMapper detectionMapper;
    private final NotificationMapper notificationMapper;
    private final LiveDataServiceGrpc.LiveDataServiceStub stub;
    private LiveDataServiceImpl liveDataService;

    // ... constructor injecting the mappers and a stub built from your gRPC ManagedChannel ...

    @Produces
    @ApplicationScoped
    public LiveDataService produceLiveDataService() {
        if (liveDataService == null) {
            liveDataService = new LiveDataServiceImpl(telemetryMapper, detectionMapper,
                notificationMapper, stub);
        }
        return liveDataService;
    }

    @Shutdown
    void shutdown() {
        if (liveDataService != null) {
            liveDataService.shutdown();
        }
    }
}
```

`LiveDataServiceImpl.shutdown()` closes every stream of all three kinds and shuts down the internal
reconnect scheduler — it's the one method that gives you a complete, orderly shutdown in one call.
`@Shutdown` here is Quarkus's own runtime shutdown-event annotation (`io.quarkus.runtime.Shutdown`),
not a CDI `@PreDestroy`.

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

5. **Keep the task id stable.** Use the same `taskId` across every `TaskEventData` notification for one run, so the platform can follow that run's progress through to completion.

6. **Do not manually manage streams.** Let the SDK handle reconnection. If you need to reset a stream, call the relevant `close*Stream(deviceSn)` and the next `produce*` call will create a new one automatically.

7. **Call `LiveDataServiceImpl.shutdown()` from your own shutdown hook.** The SDK will not do this for you — see [Wiring and shutdown](#wiring-and-shutdown). Do it before tearing down the gRPC channel the stub was built from.
