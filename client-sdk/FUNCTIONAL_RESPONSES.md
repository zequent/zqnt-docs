# Zequent Client SDK — Functional Responses

Every RemoteControl call returns a response as soon as the edge adapter replies — but "success" in that response is easy to misread as "the device did the thing." This page explains what a command response actually represents, and how to confirm what really happened on the device.

For Python, see [FUNCTIONAL_RESPONSES_PYTHON.md](FUNCTIONAL_RESPONSES_PYTHON.md).

## Command calls are synchronous, not fire-and-forget

`client.remoteControl().takeoff(request)` doesn't return the moment your call is accepted — it blocks (behind the `CompletableFuture`) until the device's own command handler has actually responded. There's no separate "ack" that arrives later on some other channel: the `CompletableFuture` you get back *is* that answer.

## What `success` actually means

A successful response means the adapter's handler for that command returned without error — nothing more. It's entirely up to the adapter's own implementation whether that means "the flight controller accepted the takeoff command" or "the drone has physically left the ground." The platform doesn't distinguish the two, and neither does the response:

```java
TakeoffResponse response = client.remoteControl().takeoff(request).get();

if (response.isSuccess()) {
    // The adapter accepted and initiated the command.
    // This does NOT mean the drone is airborne yet.
    System.out.println(response.getMessage()); // e.g. "Takeoff initiated"
} else {
    System.out.println(response.getError().getErrorMessage());
}
```

Treat a successful response as "the command was handed off and accepted," not as confirmation of the physical outcome. For that, you need telemetry.

**What that response actually looks like**, e.g. printed via `System.out.println(response)`:

```
TakeoffResponse(success=true, message=Takeoff initiated, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=550e8400-e29b-41d4-a716-446655440000, error=null, progress=null)
```

A rejected command looks the same shape, with `error` populated instead of `message`:

```
TakeoffResponse(success=false, message=null, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=550e8400-e29b-41d4-a716-446655440000, error=ErrorInfo(errorCode=ERROR_CODE_ASSET, errorMessage=Asset ZQT-DOCK-0417 is not connected, timestamp=2026-08-26T18:41:52), progress=null)
```

## Confirming what actually happened, via telemetry

The response to a command call isn't the source of truth for device state — the telemetry stream is. Subscribe with `client.liveData()` and watch the relevant field, rather than assuming the command response means the job is done:

| Command(s) | Field to watch | Confirms it worked |
| --- | --- | --- |
| `takeoff()` | `SubAssetTelemetry.mode` | reaches `SUBASSET_MODE_TAKEOFF_FINISHED` (after `TAKEOFF_PREPARE` → `TAKEOFF_AUTO`) |
| `openCover()` / `closeCover()` | `AssetTelemetry.coverState` | becomes `COVER_STATE_OPENED` / `COVER_STATE_CLOSED` |
| `startCharging()` / `stopCharging()` | `AssetTelemetry.subAssetCharging` | becomes `true` / `false` |
| `enterManualControl()` / `exitManualControl()` | `AssetTelemetry.hasActiveManualControlSession` | becomes `true` / `false` |
| `returnToHome()` | `SubAssetTelemetry.mode` | moves through `RETURN_AUTO` → `LANDING_AUTO` → back to `IDLE` once docked |

Worked example, using `takeoff()`:

```java
client.remoteControl().takeoff(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            System.out.println("Adapter rejected takeoff: " + response.getError().getErrorMessage());
            return;
        }

        // Command was accepted — now watch telemetry for what actually happens.
        var telemetryRequest = new StreamTelemetryRequest();
        telemetryRequest.setSn(request.getSn());

        client.liveData().streamTelemetryData(telemetryRequest, response -> {
            TelemetryData data = response.getTelemetry();
            if (data == null) {
                return;   // heartbeat, source-status or error frame — check response.getEventType()
            }
            // This asset is a dock, so the drone's own readings arrive as SUB_ASSET frames.
            if (data.getSourceType() == TelemetryData.SourceType.SUB_ASSET
                    && data.getSubAsset().getMode() == SubAssetMode.SUBASSET_MODE_TAKEOFF_FINISHED) {
                System.out.println("Drone is airborne.");
            }
        });
    });
```

### The shape of a telemetry frame

`StreamTelemetryResponse` carries a single `TelemetryData telemetry` object. Inside it, exactly one
of `asset` / `subAsset` is populated, and `getSourceType()` is **derived** from which one that is:

| `getSourceType()` | `asset` | `subAsset` | The reading describes |
| --- | --- | --- | --- |
| `ASSET` | populated | `null` | the registered top-level Asset itself |
| `SUB_ASSET` | `null` | populated | a child SubAsset belonging to that Asset |

Shared position fields (`latitude`, `longitude`, `absoluteAltitude`, `relativeAltitude`,
`windSpeed`, `heading`) live directly on `TelemetryData`, not duplicated per source.

**An Asset is not necessarily a dock.** An Asset is whatever top-level entity was registered — a
drone, a dock, a ground vehicle, a sensor gateway, a camera. A drone operated independently is
registered as an Asset in its own right; a drone that belongs to a dock is reported as that dock's
SubAsset. Both are supported, and both are shown below. See
[Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) for the full model.

**Example A — a standalone drone, registered directly as an Asset.** No parent dock exists, so
`subAsset` is `null` and the source type is `ASSET`:

```
StreamTelemetryResponse(
    tid=8f14e45f-ceea-467a-9575-9f2a1b0c3d4e,
    timestamp=2026-08-26T18:41:57.385Z,
    hasErrors=false,
    eventType=TELEMETRY,
    sn=SIM-DRONE-001,
    assetId=c1d7f3a2-95b4-4c1e-8f6d-2a7b9e0c4513,
    telemetry=TelemetryData(
        id=SIM-DRONE-001,
        sn=SIM-DRONE-001,
        timestamp=2026-08-26T18:41:57.385,
        latitude=52.52000045776367,
        longitude=13.404999732971191,
        absoluteAltitude=34.0,
        relativeAltitude=34.0,
        windSpeed=3.305775,
        heading=0.0,
        asset=AssetDetails(
            mode=ASSET_MODE_IDLE,
            subAssetPercentage=100.0,
            hasActiveManualControlSession=false,
            positionValid=true,
            positionState=PositionState(gpsNumber=14, rtkNumber=0, quality=5),
            networkInformation=NetworkInformation(type=NETWORK_TYPE_4_G, rate=12.4, quality=NETWORK_STATE_QUALITY_GOOD),
            manualControlState=MANUAL_CONTROL_STATE_DISCONNECTED,
            environmentTemp=21.5, humidity=48.0, rainfall=RAINFALL_NO,
            coverState=null, airConditioner=null, insideTemp=null   // dock-only hardware
        ),
        subAsset=null
    ),
    error=null
)
// telemetry.getSourceType() == SourceType.ASSET
```

**Example B — a drone that is a SubAsset of a dock**, mid-takeoff. Here `sn` is the *parent dock's*
serial, while `telemetry.getId()` identifies the drone the reading is about:

```
StreamTelemetryResponse(
    tid=f47ac10b-58cc-4372-a567-0e02b2c3d479,
    timestamp=2026-08-26T18:42:07.912Z,
    hasErrors=false,
    eventType=TELEMETRY,
    sn=ZQT-DOCK-0417,
    assetId=550e8400-e29b-41d4-a716-446655440000,
    telemetry=TelemetryData(
        id=ZQT-DRONE-1123,
        sn=ZQT-DRONE-1123,
        timestamp=2026-08-26T18:42:07.912,
        latitude=41.015137,
        longitude=28.979530,
        absoluteAltitude=42.6,
        relativeAltitude=12.4,
        windSpeed=4.6,
        heading=187.5,
        asset=null,
        subAsset=SubAssetDetails(
            horizontalSpeed=2.8,
            verticalSpeed=1.1,
            windDirection=NORTH_WEST,
            gear=1,
            mode=SUBASSET_MODE_TAKEOFF_AUTO,
            country=TR,
            heightLimit=120,
            homeDistance=18.4,
            batteryInformation=BatteryInformation(percentage=78, remainingTime=1320, returnToHomePower=22)
        )
    ),
    error=null
)
// telemetry.getSourceType() == SourceType.SUB_ASSET
```

Note `BatteryInformation.percentage` and `returnToHomePower` are `String`, not numbers.

Every telemetry frame carries exactly one of `asset` or `subAsset` — never both, never neither.
A frame with `telemetry == null` is not a telemetry reading at all: check `getEventType()` for
`HEARTBEAT`, `SOURCE_STATUS` or `ERROR`.

See [Connector](CONNECTOR.md) for looking up asset state on demand instead of streaming it, and [Quickstart](QUICKSTART.md) for how `client.remoteControl()` / `client.liveData()` get wired up in the first place.
