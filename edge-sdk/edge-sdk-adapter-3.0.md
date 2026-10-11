# Zequent Edge SDK (Java) — Adapters on 3.0 (preview)

> **Preview.** This page describes the **3.0** adapter API of the Java Edge SDK. 3.0 is not
> released yet: the API below is available as a preview build. Names may still change before the
> release. For the current 2.0 line, see [Edge Adapter (Java)](edge-sdk-adapter.md) and the
> [Java Quickstart](edge-sdk-quickstart.md).

On 3.0 an adapter writes each command **once**: a dotted command id, a JSON Schema for its params,
a JSON Schema for its result and a handler. Everything else is derived from that registration:
what the asset advertises (`GetCapabilities`, v2 and v3), the v3 `ExecuteCommand` the platform
calls, and the v2 typed RPCs a 2.x platform still calls. An adapter built this way works against
2.x and 3.0 platforms alike.

Python: [Adapters on 3.0 (Python)](edge-sdk-python-adapter-3.0.md) · Go:
[Adapters on 3.0 (Go)](edge-sdk-go-adapter-3.0.md) · Platform: [Upgrading the platform to 3.0](../concepts/upgrading-platform-to-3.0.md).

## Step 1: Add the preview dependency

The 3.0 preview is published to GitHub Packages as a `-SNAPSHOT` build. Every push to the preview
branch publishes a new version `3.0.0-PR<n>-<sha>-SNAPSHOT`; at the time of writing:

```xml
<dependency>
    <groupId>com.zqnt.sdk</groupId>
    <artifactId>edge-java-sdk</artifactId>
    <!-- preview, not a release -->
    <version>3.0.0-PR9-bce04b5-SNAPSHOT</version>
</dependency>
```

The SDK's generated types and shared classes now come from **`com.zqnt:zqnt-api-java`** (public,
Apache 2.0), which replaces `zqnt-utils-java`. The Java packages are unchanged
(`com.zqnt.utils.*`, `com.zqnt.protos.*`), so no import changes. Both are on GitHub Packages; your
`~/.m2/settings.xml` needs a token with package read access for:

```xml
<repository>
    <id>zqnt-edge-sdk-repo</id>
    <url>https://maven.pkg.github.com/Zequent/zqnt-edge-sdk-java</url>
</repository>
<repository>
    <id>zqnt-api-repo</id>
    <url>https://maven.pkg.github.com/Zequent/zqnt-api-java</url>
</repository>
```

Move to the released `3.0.0` once it is out; do not run a preview in production. Configuration and
authentication (`ZQNT_EDGE_TOKEN`, `ZQNT_PLATFORM_PUBLIC_KEY`, `ZQNT_EDGE_AUTH_DISABLED`) are
unchanged from 2.0 — see [Edge SDK Configuration](edge-sdk-configuration.md).

## Step 2: Declare the commands

Extend `RegistryEdgeAdapter` and register every command in the constructor:

```java
import com.zqnt.protos.capability.v3.CompletionMode;
import com.zqnt.protos.capability.v3.TelemetryField;
import com.zqnt.protos.capability.v3.TelemetryValueType;
import com.zqnt.sdk.edge.adapter.application.RegistryEdgeAdapter;
import com.zqnt.sdk.edge.adapter.domains.Capability;
import com.zqnt.sdk.edge.adapter.domains.CommandResult;
import com.zqnt.utils.common.proto.AssetTypeEnum;

import java.util.Map;
import java.util.concurrent.CompletableFuture;

public class AcmeDrone extends RegistryEdgeAdapter {

    public AcmeDrone(AcmeLink link) {
        super(AssetTypeEnum.ASSET_TYPE_AIRCRAFT);

        Capability takeoff = registerCommand("flight.takeoff",
                Map.of("type", "object", "properties", Map.of(
                        "altitude", Map.of("type", "number", "minimum", 2, "maximum", 120))),
                Map.of(),
                request -> link.takeOff(request.params())
                        .thenApply(flightId -> CommandResult.accepted("climbing", flightId, request.sn())));
        takeoff.setCompletion(CompletionMode.COMPLETION_MODE_ASYNCHRONOUS);
        takeoff.setCompletionEvent("flight.takeoff.completed");

        registerCommand("vendor.acme.spray",
                Map.of("type", "object", "required", java.util.List.of("seconds"),
                        "properties", Map.of("seconds", Map.of("type", "integer", "minimum", 1))),
                Map.of(),
                request -> link.spray((Integer) request.params().get("seconds"))
                        .thenApply(done -> CommandResult.success("sprayed", request.sn())));

        commands().declareTelemetryField(TelemetryField.newBuilder().setKey("vendor.acme.tank_level")
                .setType(TelemetryValueType.TELEMETRY_VALUE_TYPE_NUMBER).setUnit("%").build());
    }
}
```

- **Ids.** Reuse the platform's ids for shared actions (`flight.takeoff`, `navigation.go_to`,
  `dock.open_cover`, … — the command catalog, `COMMAND_CATALOG.md` next to the v3 contracts in the
  public `zqnt-api-java` repository). A device-specific command goes under
  `vendor.<vendor>.<name>`.
- **Handler.** A `CommandHandler` gets a `CommandRequest`: `sn()`, `targetRef()` (sub-asset, payload
  or component; `null` for the asset itself), `commandId()`, `params()` and `commandExecutionId()`
  (the platform's id of this run on v3, `null` when the command came over v2).
- **Validation.** Params are validated against the input schema **before** the handler runs;
  numbers become `Integer`/`Long` where the schema says `integer`. Invalid params are `REJECTED`
  with error code `command.invalid_params` (`CommandRegistry.INVALID_PARAMS`) and a readable
  message; an id that is not registered is `REJECTED` with `command.not_supported`. A schema the SDK
  does not understand fails `registerCommand` with an `IllegalArgumentException`.
- **Result.** `CommandResult.success(...)` (finished, optionally with an output map),
  `CommandResult.accepted(message, externalExecutionId, sn)` (still running, the outcome follows as a
  command event — Step 4), `CommandResult.rejected(errorCode, message, sn)`, `CommandResult.error(...)`.
- **Completion.** `setCompletion(COMPLETION_MODE_ON_REPLY)` says the reply is the outcome;
  `COMPLETION_MODE_ASYNCHRONOUS` with `setCompletionEvent(...)` says a later event is. Unset, the
  platform goes by the result: an accepted result waits, any other success is done.
- **Changing what the asset can do.** `registerCommand` again replaces an id, `commands().unregister(id)`
  withdraws it; after changing a returned `Capability` later (its `state`, e.g.
  `CAPABILITY_STATE_TEMPORARILY_UNAVAILABLE` while a payload is detached), call `commands().changed()`
  so the platform learns about it.

The typed methods of `EdgeAdapterService` (`takeOff`, `openCover`, …) are deprecated. In a
`RegistryEdgeAdapter` they are final and route onto the registered handler of the matching id
(`takeOff` → `flight.takeoff`); omitted coordinates (NaN) become absent params. Adapters that still
implement the typed methods keep working.

### Command risk (`CommandSafety`)

The 3.0 contract rates every command with a `CommandSafety` (`COMMAND_RISK_OBSERVE`, `_ADJUST`,
`_MOVE`, `_CRITICAL`, plus reversibility and an undo command). The 3.0 SDK previews **do not let
an adapter set it**: the platform rates a command by the command catalog, and an id that is not in
the catalog — every `vendor.*` command — counts as **critical** (a client must give a reason to run
it; approvals and agent policy treat it accordingly).

## Step 3: Serve both contracts

Serve `EdgeAdapterGrpcServiceImpl` (v2) and `EdgeAdapterV3GrpcServiceImpl` (v3) from the same
adapter, behind the `PlatformAuthServerInterceptor` that only accepts commands signed by the
platform. In Quarkus, as `@GrpcService` subclasses; with plain grpc-java:

```java
import com.zqnt.sdk.edge.adapter.grpc.EdgeAdapterGrpcServiceImpl;
import com.zqnt.sdk.edge.adapter.grpc.EdgeAdapterV3GrpcServiceImpl;
import com.zqnt.sdk.edge.application.ProtoJsonMapper;
import com.zqnt.sdk.edge.auth.EdgeAuthConfig;
import com.zqnt.sdk.edge.auth.PlatformAuthServerInterceptor;
import io.grpc.ServerBuilder;
import io.grpc.ServerInterceptors;

AcmeDrone adapter = new AcmeDrone(link);
PlatformAuthServerInterceptor auth = new PlatformAuthServerInterceptor(EdgeAuthConfig.fromEnv());

ServerBuilder.forPort(9090)
        .addService(ServerInterceptors.intercept(new EdgeAdapterGrpcServiceImpl(adapter, new ProtoJsonMapper()), auth))
        .addService(ServerInterceptors.intercept(new EdgeAdapterV3GrpcServiceImpl(adapter), auth))
        .build()
        .start();
```

A 3.0 platform calls v3 `ExecuteCommand`; a 2.x platform calls the v2 RPCs. Both run the same
handler.

## Step 4: Talk to the platform

Calls into the platform go over two channels, both carrying the edge credential
(`EdgeCredentialsClientInterceptor`): the **remote-control** channel (`EdgeGatewayService`:
capability reports, command events) and the **live-data** channel (telemetry).

```java
import com.zqnt.sdk.edge.gateway.CapabilityReporter;
import com.zqnt.sdk.edge.gateway.EdgeGatewayClient;
import com.zqnt.sdk.edge.livedata.application.impl.LiveDataServiceImpl;
import com.zqnt.sdk.edge.telemetry.TelemetryPublisher;

EdgeGatewayClient gateway = new EdgeGatewayClient(remoteControlChannel);
LiveDataService liveData = new LiveDataServiceImpl(telemetryMapper, detectionMapper, notificationMapper,
        LiveDataServiceGrpc.newStub(liveDataChannel), gateway);
new CapabilityReporter(adapter, gateway).track("SN-1");
TelemetryPublisher telemetry = new TelemetryPublisher(liveDataChannel, liveData);
```

| What | 3.0 platform | 2.x platform (answers `UNIMPLEMENTED`) |
| --- | --- | --- |
| Capabilities | `CapabilityReporter.track(sn)`: `ReportCapabilities` at start and on every registry change, with the declared telemetry fields; retried until it succeeds | v2 `ReportAssetRuntime`, without telemetry fields |
| Command progress and outcome | the events you already publish (Step 5) go to `PublishCommandEvent` under the platform's `command_execution_id` | v2 notification stream, as before |
| Telemetry, detections, alerts | `TelemetryPublisher.publish(TelemetrySample / DetectionBatch / Alert)`: one long-lived stream per kind, reconnected with backoff | v2 `ProduceTelemetry` / `ProduceDetection` with the shared fields only — see Step 6 |

**Automatic fallback.** After an `UNIMPLEMENTED` answer the SDK stays on v2 for **10 minutes**
(`V3Availability.DEFAULT_RETRY_AFTER`), then tries v3 again — an adapter picks up a platform upgrade
without a restart. `TelemetryPublisher.v3Available()` tells which path is in use. An adapter that
reports detected payloads in its v2 runtime snapshot passes its own `V2CapabilityReport` to
`new EdgeGatewayClient(remoteControlChannel, v2Report)`, because `ReportAssetRuntime` replaces the
whole snapshot.

`CapabilityReporter.track(sn)` once per asset the adapter serves (`untrack(sn)` to stop).

## Step 5: Report command events

A command that answered `accepted(...)` reports its progress and outcome as
`CommandExecutionEventData`, through `LiveDataService.produceNotificationData(...)` exactly as on
2.0 (see [Edge Live Data](edge-sdk-live-data.md)):

```java
NotificationRequestData done = NotificationRequestData.builder()
    .sn("SN-1")
    .timestamp(LocalDateTime.now())
    .eventType(NotificationEventType.NOTIFICATION_EVENT_COMMAND_EXECUTION)
    .commandExecutionEvent(CommandExecutionEventData.builder()
        .externalExecutionId(flightId)          // the id you returned in CommandResult.accepted(...)
        .commandId("flight.takeoff")
        .assetSn("SN-1")
        .status(CommandExecutionStatus.COMMAND_EXECUTION_STATUS_SUCCEEDED)
        .occurredAt(LocalDateTime.now())
        .build())
    .build();
liveData.produceNotificationData(done);
```

With an `EdgeGatewayClient` on the `LiveDataServiceImpl`, the SDK sends it as v3
`PublishCommandEvent` and puts the platform's `command_execution_id` on it, found again from your
`externalExecutionId` (or by asset and command). Every event carries `occurred_at` — now, when you
gave none. Finish every accepted command with exactly one terminal status (`SUCCEEDED`, `FAILED`,
`CANCELLED`), or a Skill waiting on it never moves on.

## Step 6: Publish telemetry

A 3.0 `TelemetrySample` has the fields every asset shares; everything device-specific goes into
`details`, under keys the asset declares (Step 2):

```java
import com.google.protobuf.util.Timestamps;
import com.zqnt.protos.common.v3.AssetRef;
import com.zqnt.protos.common.v3.GeoPoint;
import com.zqnt.protos.telemetry.v3.TelemetrySample;
import com.zqnt.sdk.edge.support.Structs;
import com.zqnt.utils.telemetry.TelemetrySampleMapper;

telemetry.publish(TelemetrySample.newBuilder()
        .setAsset(AssetRef.newBuilder().setSn("SN-1"))
        .setObservedAt(Timestamps.now())
        .setPosition(GeoPoint.newBuilder().setLatitude(47.5).setLongitude(9.7).setAltitude(420.0))
        .setRelativeAltitude(40.0)
        .setBatteryPercent(81.0)
        .setDetails(Structs.toStruct(Map.of(
                TelemetrySampleMapper.DRONE_GEAR, 1,
                "vendor.acme.tank_level", 63.0)))
        .build());
```

`observed_at` defaults to now. Altitude in `position` is above sea level, `relative_altitude` above
the takeoff point.

**Standard keys.** `TelemetrySampleMapper` (in `zqnt-api-java`) holds the platform's **telemetry
catalog**: 64 detail keys with their type, unit and allowed values — the ones listed in
[The telemetry catalog](#the-telemetry-catalog) below. `TelemetrySampleMapper.telemetryFields()`
returns their `TelemetryField` descriptions; declare the ones your asset sends with
`commands().declareTelemetryField(...)`, so their type and unit are the platform's. Use your own
`vendor.<vendor>.<name>` keys for everything else.

**On a 2.x platform** the Java SDK sends only the shared fields (position, altitudes, heading,
speeds, battery) over v2: a sample's `details` and all alerts are dropped. The Python and Go SDKs
map catalog keys onto their v2 fields; the Java preview does not yet.

### The telemetry catalog

| Area | Keys |
| --- | --- |
| Weather | `wind.speed` (m/s), `wind.direction` |
| Dock | `dock.environment_temperature` (°C), `dock.inside_temperature` (°C), `dock.humidity` (%), `dock.mode`, `dock.rainfall`, `dock.cover_state`, `dock.debug_mode_open`, `dock.position_valid`, `dock.working_voltage` (mV), `dock.working_current` (mA), `dock.supply_voltage` (V) |
| Dock air conditioner, manual control | `dock.air_conditioner.state`, `dock.air_conditioner.switch_time` (s), `dock.manual_control.state`, `dock.manual_control.active_session` |
| Docked aircraft | `dock.drone.sn`, `dock.drone.model`, `dock.drone.paired`, `dock.drone.online`, `dock.drone.at_home`, `dock.drone.charging` |
| Dock positioning | `dock.gnss.gps_satellites`, `dock.gnss.rtk_satellites`, `dock.gnss.quality` |
| Network and links | `network.type`, `network.rate` (KB/s), `network.quality`, `link.4g.frequency_band`, `link.4g.ground_quality`, `link.4g.link_state`, `link.4g.quality`, `link.4g.aircraft_quality`, `link.dongle_count`, `link.work_mode`, `link.sdr.frequency_band` (GHz), `link.sdr.link_state`, `link.sdr.quality`, `sdr.down_quality`, `sdr.up_quality`, `sdr.frequency_band` (GHz) |
| Aircraft | `drone.mode`, `drone.gear`, `drone.height_limit` (m), `drone.home_distance` (m), `drone.total_flight_distance` (m), `drone.total_flight_time` (s), `drone.country`, `drone.battery.remaining_time` (s), `drone.battery.return_to_home_power` |
| Payload and camera | `payload.id`, `payload.name`, `payload.observed_at` (RFC 3339), `camera.current_lens`, `camera.gimbal_pitch` (°), `camera.gimbal_yaw` (°), `camera.gimbal_roll` (°), `camera.zoom_factor` |
| Range finder, sensor | `range_finder.target_latitude` (°), `range_finder.target_longitude` (°), `range_finder.target_distance` (m), `range_finder.target_altitude` (m), `sensor.target_temperature` (°C) |

Each key's type follows its 2.x field: numbers, booleans and strings. Enumerations (`dock.mode`,
`dock.cover_state`, `drone.mode`, …) are strings — the value's 2.x name without its enum prefix —
and their `TelemetryField` lists the `allowed_values`. The
same catalog is in `zqnt_api.telemetry` (Python) and `github.com/zequent/zqnt-api-go/v3/telemetry`
(Go).

## Pairing with a claim code

Pairing is unchanged from 2.0: `ConnectorService.ensureAsset(asset, claimCode)` binds to an asset the
platform already knows and redeems a one-time **claim code** (`ZQNT_CLAIM_CODE`, minted in the
console) only for an unknown serial; `redeemAssetClaim(code, asset)` and `describeAssetClaim(code)`
are there for custom flows. See [Edge Connector — pairing](edge-sdk-connector.md).

`registerAsset` is deprecated: it creates an asset without an organization. Removing it, together
with the platform's organization-UUID bind, is **planned** for 3.0; pair with claim codes.

## Step 7: Check the adapter with the conformance kit

```java
import com.zqnt.sdk.edge.conformance.EdgeAdapterConformance;

@Test
void conformsToTheEdgeContract() {
    EdgeAdapterConformance.check(new AcmeDrone(fakeLink), "SN-1").assertPassed();
}
```

`check` verifies that every advertised id is executable and every executable id is advertised
(it probes unregistered ids, so fake the device side), that the schemas parse, and that an
`ASYNCHRONOUS` command declares its completion event. `EdgeAdapterConformance.checkCommandEvents(...)`
and `checkCommandExecutionEvents(sn, ...)` check recorded events for execution id, asset, state and
`occurred_at`. A `ConformanceReport` lists `problems()`; `passed()` / `assertPassed()` decide.
