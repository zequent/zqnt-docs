# Zequent Client SDK — Functional Responses

A Remote Control call answers once the platform has checked your command and the asset's edge
adapter has replied — but "success" in that answer is easy to misread as "the device did the
thing." This page explains what a command response represents, and how to confirm what really
happened on the device.

For Python, see [FUNCTIONAL_RESPONSES_PYTHON.md](FUNCTIONAL_RESPONSES_PYTHON.md); for Go,
[FUNCTIONAL_RESPONSES_GO.md](FUNCTIONAL_RESPONSES_GO.md).

## What happens when you send a command

Every command except manual control runs as a **single-command run** — the same execution engine
that runs [Applications & Skills](../concepts/applications-and-skills.md):

1. **The platform checks it.** A movement command meets your organization's no-fly zones: a route
   through a zone is flown around it, a command with no possible detour is refused, and a zone that
   requires approval holds the command until someone approves it (see
   [No-fly zones and safe returns](../concepts/airspace-safety.md)). Pre-flight checks your
   organization has switched on can refuse it too.
2. **The platform dispatches it** to the asset's edge adapter.
3. **Your call answers** once the adapter has replied. The `CompletableFuture` you get back *is*
   that answer; there is no separate acknowledgement on another channel.

## What the response tells you

| Response | Meaning |
| --- | --- |
| `success == false` | The command was refused before it reached the device (no-fly zone without a detour, a failed pre-flight check), or the adapter rejected it. `error.errorMessage` says why |
| `success == true`, `progress.state == "SKILL_EXECUTION_STATUS_SUCCEEDED"` | The adapter carried out a command that completes at once — dock, camera, charging, reboot and similar commands |
| `success == true`, `progress.state == "SKILL_EXECUTION_STATUS_RUNNING"` | The command is under way. `takeoff`, `goTo`, `lookAt` and `returnToHome` stay running until the adapter reports their outcome; a command waiting for an approval is running too |

For a running command the outcome arrives later. Follow it with
`client.missionAutonomy().listSkillExecutions(...)`, filtered by the asset's serial number (see
[Tracking and controlling an execution](../concepts/applications-and-skills.md#tracking-and-controlling-an-execution)),
or watch telemetry as shown below.

Even `SUCCEEDED` means the adapter reported the command as done — whether that is "the flight
controller accepted it" or "the cover is physically open" depends on the adapter. Telemetry is the
source of truth for device state.

```java
TakeoffResponse response = client.remoteControl().takeoff(request).get();

if (!response.isSuccess()) {
    System.out.println("Refused: " + response.getError().getErrorMessage());
} else {
    // e.g. SKILL_EXECUTION_STATUS_RUNNING: the drone is taking off, not airborne yet
    System.out.println(response.getProgress().getState());
}
```

**What that response looks like**, printed via `System.out.println(response)`:

```
TakeoffResponse(success=true, message=null, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=null, error=null, progress=ProgressInfo(progress=0.0, state=SKILL_EXECUTION_STATUS_RUNNING, leftTimeInSeconds=0.0))
```

A refused command carries `error` instead of `progress`:

```
TakeoffResponse(success=false, message=null, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=null, error=ErrorInfo(errorCode=ERROR_CODE_CLIENT, errorMessage=Refused: The destination lies inside no-fly zone 'Airport'., timestamp=2026-08-26T18:41:52), progress=null)
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
            System.out.println("Takeoff refused: " + response.getError().getErrorMessage());
            return;
        }

        // The takeoff is running — watch telemetry for what actually happens.
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
