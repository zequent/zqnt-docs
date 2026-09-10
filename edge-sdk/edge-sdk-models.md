# Edge SDK -- Models Reference

This document provides a comprehensive reference for all request, response, and data models used in the Zequent Edge SDK.

## Table of Contents

- [Command Models](#command-models)
  - [CommandResult](#commandresult)
  - [CurrentCapabilities](#currentcapabilities)
  - [Capability](#capability)
- [Flight Control Models](#flight-control-models)
  - [TakeOffRequest](#takeoffrequest)
  - [ReturnToHomeRequest](#returntohomerequest)
  - [GoToRequest](#gotorequest)
  - [Coordinates](#coordinates)
- [Manual Control Models](#manual-control-models)
  - [ManualControlRequest](#manualcontrolrequest)
  - [ManualControlInput](#manualcontrolinput)
- [Camera and Gimbal Models](#camera-and-gimbal-models)
  - [LookAtRequest](#lookatrequest)
  - [ChangeLensRequest](#changelensrequest)
  - [ChangeZoomRequest](#changezoomrequest)
- [Live Streaming Models](#live-streaming-models)
  - [LiveStreamStartRequest](#livestreamstartrequest)
  - [LiveStreamStopRequest](#livestreamstoprequest)
- [Telemetry Models](#telemetry-models)
  - [TelemetryRequestData](#telemetryrequestdata)
  - [TelemetryData](#telemetrydata)
  - [AssetDetails](#assetdetails)
  - [SubAssetDetails](#subassetdetails)
- [Mission Models](#mission-models)
  - [MissionData](#missiondata)
- [Configuration Models](#configuration-models)
  - [EdgeClientConfig](#edgeclientconfig)

---

## Command Models

### CommandResult

The universal return type for all `EdgeAdapterService` commands.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `success` | `boolean` | Whether the command succeeded |
| `message` | `String` | Human-readable result message |
| `tid` | `String` | Transaction ID for tracing |
| `sn` | `String` | Device serial number |
| `resultType` | `CommandResultType` | Result classification |
| `externalExecutionId` | `String` | Vendor-assigned id for a command still running asynchronously (e.g. a DJI `flightId`). Set via `accepted(...)`; used to correlate async progress events and route cancellation back to the right execution. |

**CommandResultType enum:**

| Value | Description |
|-------|-------------|
| `SUCCESS` | Command executed successfully |
| `ACCEPTED` | Command was accepted and is still running asynchronously — track it via `externalExecutionId` |
| `ERROR` | Command failed |
| `NOT_IMPLEMENTED` | Command is not supported by this adapter |

**Factory methods:**

```java
// Success
CommandResult.success("Message", sn)
CommandResult.success("Message", tid, sn)

// Accepted — still running asynchronously
CommandResult.accepted("Message", externalExecutionId, sn)

// Error
CommandResult.error("Error message", sn)
CommandResult.error("Error message", tid, sn)

// Not implemented (used internally by default methods)
CommandResult.notImplemented("Not supported", sn)
```

**Utility methods:**

```java
boolean isNotImplemented()  // returns true if resultType == NOT_IMPLEMENTED
boolean isAccepted()        // returns true if resultType == ACCEPTED
```

---

### CurrentCapabilities

Reports the set of capabilities that an adapter currently supports.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `assetType` | `AssetTypeEnum` | Type of asset |
| `capabilities` | `Set<Capability>` | Set of supported capabilities |
| `timestamp` | `long` | Timestamp when capabilities were reported |

**Factory methods:**

```java
// Empty capabilities
CurrentCapabilities.empty(sn)

// With specific capabilities
CurrentCapabilities.of(sn, AssetTypeEnum.ASSET_TYPE_DOCK, capabilitySet)
```

---

### Capability

Describes a single capability of the adapter.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `command` | `String` | Command name (e.g., "takeOff", "openCover") |
| `description` | `String` | Human-readable description |
| `available` | `Boolean` | Whether the capability is currently available |
| `unavailableReason` | `String` | Reason if not available (null if available) |
| `metadata` | `Map<String, String>` | Additional metadata key-value pairs |

---

## Flight Control Models

### TakeOffRequest

Request to initiate takeoff.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `coordinates` | `Coordinates` | Takeoff position (latitude, longitude, altitude) |

---

### ReturnToHomeRequest

Request to return the sub-asset (drone) to its home position.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `tid` | `String` | Transaction identifier |
| `sn` | `String` | Device serial number |
| `altitude` | `Float` | Return altitude in meters |

---

### GoToRequest

Request to navigate to specific coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `coordinates` | `Coordinates` | Target position |

---

### Coordinates

Geographic coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `latitude` | `Float` | GPS latitude in degrees |
| `longitude` | `Float` | GPS longitude in degrees |
| `altitude` | `Float` | Altitude in meters |

---

## Manual Control Models

### ManualControlRequest

Request to enter or exit manual control mode.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `enable` | `boolean` | `true` to enter manual control, `false` to exit |

---

### ManualControlInput

A single frame of joystick/stick input for controlling the device in real-time.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `roll` | `Float` | Roll axis value (typically -1.0 to 1.0) |
| `pitch` | `Float` | Pitch axis value (typically -1.0 to 1.0) |
| `yaw` | `Float` | Yaw axis value (typically -1.0 to 1.0) |
| `throttle` | `Float` | Throttle value (typically -1.0 to 1.0) |
| `gimbalPitch` | `Float` | Gimbal pitch adjustment |

This model supports the `@Builder` pattern:

```java
ManualControlInput input = ManualControlInput.builder()
    .sn("DEVICE_SN")
    .roll(0.0f)
    .pitch(0.5f)
    .yaw(0.0f)
    .throttle(0.3f)
    .gimbalPitch(-0.2f)
    .build();
```

---

## Camera and Gimbal Models

### LookAtRequest

Request to point the camera at specific coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `latitude` | `Double` | Target GPS latitude |
| `longitude` | `Double` | Target GPS longitude |
| `altitude` | `Float` | Target altitude in meters |
| `locked` | `Boolean` | Whether to lock the gimbal on the target |
| `payloadIndex` | `String` | Camera/payload index identifier |

---

### ChangeLensRequest

Request to switch the active camera lens.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `lens` | `String` | Target lens identifier (e.g., "wide", "zoom", "ir") |
| `videoId` | `String` | Video stream identifier |

---

### ChangeZoomRequest

Request to adjust the camera zoom level.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `lens` | `String` | Lens identifier |
| `payloadIndex` | `String` | Payload index |
| `zoom` | `Float` | Target zoom factor |

---

## Live Streaming Models

### LiveStreamStartRequest

Request to start a video live stream.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `videoId` | `String` | Video stream identifier |
| `streamServer` | `String` | Target streaming server URL |
| `videoType` | `String` | Stream type (e.g., RTMP, RTSP) |

---

### LiveStreamStopRequest

Request to stop a video live stream.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `videoId` | `String` | Video stream identifier to stop |

---

## Telemetry Models

### TelemetryRequestData

Top-level wrapper for pushing telemetry data to the Live Data Service.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `tid` | `String` | Transaction identifier |
| `sn` | `String` | Device serial number |
| `assetId` | `String` | Platform asset identifier |
| `timestamp` | `LocalDateTime` | When the telemetry was recorded |
| `telemetry` | `TelemetryData` | The actual telemetry payload — see below |

Supports the `@Builder` pattern:

```java
TelemetryRequestData data = TelemetryRequestData.builder()
    .sn("DEVICE_SN")
    .tid(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .telemetry(telemetryData)
    .build();
```

---

### TelemetryData

Unified asset and sub-asset telemetry. Exactly one of `asset` / `subAsset` must be set — which one determines `getSourceType()` (`ASSET`, `SUB_ASSET`, or `UNSPECIFIED` if neither/both are set). Position and movement fields that both an asset and a sub-asset can report (`latitude`, `longitude`, `absoluteAltitude`, `relativeAltitude`, `windSpeed`, `heading`) live directly on `TelemetryData`, not duplicated per source type.

`asset` and `subAsset` describe the **source/context** of the reading — which entity it is about — not a device category. An **Asset** is the registered top-level entity (a drone, dock, ground vehicle, sensor gateway, camera or station); a **SubAsset** is an optional child entity belonging to an Asset. Set `.asset(...)` when the reading describes the Asset itself, including when that Asset *is* the drone, and `.subAsset(...)` when it describes a child SubAsset. A drone registered directly as an Asset therefore correctly reports `sourceType = ASSET` with `subAsset` left `null`. See [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) for worked examples of both configurations.

**Package:** `com.zqnt.utils.edge.sdk.domains`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Telemetry identifier (required) |
| `timestamp` | `LocalDateTime` | Measurement time (required) |
| `sn` | `String` | Device serial number (required) |
| `latitude` | `Double` | GPS latitude |
| `longitude` | `Double` | GPS longitude |
| `absoluteAltitude` | `Float` | Altitude above sea level (m) |
| `relativeAltitude` | `Float` | Altitude above ground (m) |
| `windSpeed` | `Float` | Wind speed (m/s) |
| `heading` | `Float` | Heading (degrees) |
| `asset` | `AssetDetails` | Set when the reading describes the registered top-level Asset itself (a standalone drone, a dock, a vehicle, a sensor gateway, ...) |
| `subAsset` | `SubAssetDetails` | Set when the reading describes a child SubAsset belonging to an Asset (e.g. a drone docked in a station) |

`validate()` throws `IllegalArgumentException` if `id`/`timestamp`/`sn` are missing, or if `asset`/`subAsset` aren't set to exactly one.

```java
TelemetryData telemetry = TelemetryData.builder()
    .id(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .sn("DOCK-1")          // or a standalone drone's own SN, e.g. "SIM-DRONE-001"
    .latitude(47.3769)
    .longitude(8.5417)
    .absoluteAltitude(450.0f)
    .asset(assetDetails)   // or .subAsset(subAssetDetails) — exactly one
    .build();
```

---

### AssetDetails

Telemetry describing the registered top-level Asset itself. Nested under `TelemetryData.asset`.

This is a **superset** covering every kind of Asset, so not every field applies to every device. A dock populates the enclosure and climate fields (`coverState`, `airConditioner`, `insideTemp`, `workingVoltage`); a standalone drone registered as an Asset leaves those `null` and populates only what it physically has. Leaving inapplicable fields `null` is correct behaviour, not incomplete telemetry.

| Field | Type | Description |
|-------|------|-------------|
| `environmentTemp` | `Float` | Ambient temperature (C) |
| `insideTemp` | `Float` | Internal temperature (C) |
| `humidity` | `Float` | Humidity (%) |
| `mode` | `AssetMode` | Operational mode |
| `rainfall` | `RainfallEnum` | Rainfall condition |
| `subAssetInformation` | `SubAssetInformation` | Paired sub-asset info |
| `subAssetAtHome` | `Boolean` | Sub-asset at dock |
| `subAssetCharging` | `Boolean` | Sub-asset charging |
| `subAssetPercentage` | `Float` | Sub-asset battery (%) |
| `debugModeOpen` | `Boolean` | Debug mode active |
| `hasActiveManualControlSession` | `Boolean` | Manual control active |
| `coverState` | `AssetCoverStateEnum` | Cover state |
| `workingVoltage` | `Integer` | Working voltage (mV) |
| `workingCurrent` | `Integer` | Working current (mA) |
| `supplyVoltage` | `Integer` | Supply voltage (mV) |
| `positionValid` | `Boolean` | GPS position valid |
| `networkInformation` | `NetworkInformation` | Network info |
| `airConditioner` | `AirConditioner` | AC state |
| `manualControlState` | `ManualControlStateEnum` | DRC state |
| `positionState` | `PositionState` | GNSS fix quality |
| `wirelessLink` | `WirelessLinkInformation` | Cellular/SDR link diagnostics |
| `sdrState` | `SdrState` | SDR link diagnostics |

### SubAssetDetails

Telemetry specific to a sub-asset (drone, vehicle). Nested under `TelemetryData.subAsset`.

| Field | Type | Description |
|-------|------|-------------|
| `horizontalSpeed` | `Float` | Horizontal speed (m/s) |
| `verticalSpeed` | `Float` | Vertical speed (m/s) |
| `windDirection` | `String` | Wind direction |
| `gear` | `Integer` | Landing gear state |
| `payloadTelemetry` | `PayloadTelemetry` | Primary payload/camera data |
| `batteryInformation` | `BatteryInformation` | Battery data |
| `heightLimit` | `Integer` | Max height limit (m) |
| `homeDistance` | `Float` | Distance to home (m) |
| `totalMovementDistance` | `Double` | Total distance traveled (m) |
| `totalMovementTime` | `Double` | Total flight time (s) |
| `mode` | `SubAssetMode` | Flight mode |
| `country` | `String` | Country code |
| `componentTelemetry` | `List<ComponentTelemetryData>` | Per-component telemetry (multiple payloads/sensors) |

### Nested types

`SubAssetInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Sub-asset serial number |
| `model` | `String` | Sub-asset model name |
| `paired` | `Boolean` | Whether paired |
| `online` | `Boolean` | Whether online |

`NetworkInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `type` | `NetworkTypeEnum` | Network type (4G, 5G, etc.) |
| `rate` | `Float` | Data rate |
| `quality` | `NetworkStateQualityEnum` | Signal quality |

`AirConditioner`:

| Field | Type | Description |
|-------|------|-------------|
| `state` | `AssetAirConditionerStateEnum` | AC state |
| `switchTime` | `Integer` | Time until state switch |

`BatteryInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `percentage` | `String` | Battery percentage |
| `remainingTime` | `Integer` | Remaining time (seconds) |
| `returnToHomePower` | `String` | Power needed for RTH |

`PayloadTelemetry`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Payload identifier |
| `timestamp` | `LocalDateTime` | Measurement time |
| `name` | `String` | Payload name |
| `cameraData` | `CameraData` | Camera state |
| `rangeFinderData` | `RangeFinderData` | Rangefinder data |
| `sensorData` | `SensorData` | Sensor readings |

`ComponentTelemetryData` (one entry per physical component/payload on a sub-asset):

| Field | Type | Description |
|-------|------|-------------|
| `componentId` | `String` | Component identifier |
| `externalId` | `String` | Vendor-assigned identifier |
| `kind` | `String` | Component kind (e.g. `"camera"`) |
| `timestamp` | `LocalDateTime` | Measurement time |
| `cameraData` | `CameraData` | Camera state |
| `rangeFinderData` | `RangeFinderData` | Rangefinder data |
| `sensorData` | `SensorData` | Sensor readings |
| `attributes` | `Map<String, Object>` | Free-form additional attributes |

`CameraData`:

| Field | Type | Description |
|-------|------|-------------|
| `currentLens` | `String` | Active lens |
| `gimbalPitch` | `Float` | Gimbal pitch (degrees) |
| `gimbalYaw` | `Float` | Gimbal yaw (degrees) |
| `gimbalRoll` | `Float` | Gimbal roll (degrees) |
| `zoomFactor` | `Float` | Current zoom level |

`RangeFinderData`:

| Field | Type | Description |
|-------|------|-------------|
| `targetLatitude` | `Double` | Target latitude |
| `targetLongitude` | `Double` | Target longitude |
| `targetDistance` | `Float` | Distance to target (m) |
| `targetAltitude` | `Float` | Target altitude (m) |

`SensorData`:

| Field | Type | Description |
|-------|------|-------------|
| `targetTemperature` | `Float` | Measured temperature (C) |

`PositionState`:

| Field | Type | Description |
|-------|------|-------------|
| `gpsNumber` | `Integer` | Number of GPS satellites in view |
| `rtkNumber` | `Integer` | Number of RTK satellites in view |
| `quality` | `Integer` | Fix quality indicator |

`WirelessLinkInformation` and `SdrState` carry low-level 4G/SDR link diagnostics (frequency band, quality, link state) — populate them only if your device actually reports this level of detail; both are optional.

---

## Mission Models

### MissionData

Represents a mission returned by the Mission Autonomy Service.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `UUID` | Unique mission identifier |
| `name` | `String` | Mission name |

---

## Configuration Models

### EdgeClientConfig

The edge adapter configuration interface, mapped from `application.properties` with `@ConfigMapping(prefix = "zequent.edge")`.

**Package:** `com.zqnt.sdk.edge.config`

| Method | Return Type | Property | Default | Description |
|--------|-------------|----------|---------|-------------|
| `endpoint()` | `String` | `zequent.edge.endpoint` | -- | Adapter listen address |
| `sn()` | `String` | `zequent.edge.sn` | -- | Device serial number |
| `timeout()` | `Duration` | `zequent.edge.timeout` | `30s` | Command timeout |
| `maxRetries()` | `int` | `zequent.edge.max-retries` | `3` | Max retry attempts |
| `assetType()` | `AssetTypeEnum` | `zequent.edge.asset-type` | -- | Asset type |
| `assetVendor()` | `AssetVendor` | `zequent.edge.asset-vendor` | -- | Asset vendor |
| `assetId()` | `Optional<String>` | `zequent.edge.asset-id` | empty | Platform asset ID |

Access these values in your code via the `EdgeClient` bean:

```java
@Inject
EdgeClient edgeClient;

String sn = edgeClient.getSn();
EdgeClientConfig config = edgeClient.getConfig();
Duration timeout = config.timeout();
```

Or inject `EdgeClientConfig` directly:

```java
@Inject
EdgeClientConfig config;

String sn = config.sn();
AssetTypeEnum type = config.assetType();
```
