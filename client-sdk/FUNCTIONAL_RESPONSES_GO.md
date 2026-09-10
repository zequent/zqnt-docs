# Zequent Client SDK (Go) — Functional Responses

Every RemoteControl call returns as soon as the edge adapter replies — but that's easy to misread as "the device did the thing." This page explains what a command call actually tells you, and how to confirm what really happened on the device.

For Java, see [FUNCTIONAL_RESPONSES.md](FUNCTIONAL_RESPONSES.md). For Python, see [FUNCTIONAL_RESPONSES_PYTHON.md](FUNCTIONAL_RESPONSES_PYTHON.md).

## `error` is the only thing you check

Unlike Java/Python, there's no `success` field to read here — `remotecontrol.Client` folds the wire-level success/failure convention into a plain Go `error` for you:

```go
rc := remotecontrol.New(conn)

coordinate := &devicecontrol.GeoCoordinate{Latitude: 41.015137, Longitude: 28.979530, Altitude: 50}
resp, err := rc.TakeOff(ctx, "ZQT-DOCK-0417", coordinate)
if err != nil {
    log.Println(err)
    // e.g. "remotecontrol: TakeOff: asset ZQT-DOCK-0417 is not connected"
    return
}
```

`err == nil` means the same thing it means in Java/Python: the adapter's own `TakeOff` handler returned without error — nothing more. It's up to that handler's own implementation whether that means "accepted" or "physically airborne." The SDK's own shipped example adapter makes the distinction explicit:

```go
func (a *MyDroneAdapter) TakeOff(ctx context.Context, req *domains.TakeOffRequest) (*domains.CommandResult, error) {
    a.log.Info("TakeOff requested", "sn", req.SN, "alt", req.Coordinates.Alt)
    // TODO: send takeoff command to real hardware here.
    return domains.SuccessWithTID("takeOff accepted", req.TID, req.SN), nil
}
```

It returns success immediately, before any hardware command is even issued. "Accepted," not "completed."

**What a successful `resp` actually contains** — on the plain success path only `HasErrors` and `Meta` are set (no progress/error payload):

```
has_errors:false  meta:{tid:"a1b2c3d4-1234-5678-9abc-def012345678"  sn:"ZQT-DOCK-0417"  timestamp:{seconds:1756233727}}
```

## Confirming what actually happened, via telemetry

`StreamTelemetry` hands you the raw gRPC server-streaming client — **this SDK does not manage reconnection for you** the way the Java/Python SDKs do. You own the `Recv()` loop and redial on error yourself. Watch the relevant field, rather than assume a command's response means the job is done:

| Command(s) | Field to watch | Confirms it worked |
| --- | --- | --- |
| `TakeOff` | `SubAssetTelemetryDetails.Mode` | reaches `SUBASSET_MODE_TAKEOFF_FINISHED` (after `TAKEOFF_PREPARE` → `TAKEOFF_AUTO`) |
| `OpenCover` / `CloseCover` | `AssetTelemetryDetails.CoverState` | becomes `COVER_STATE_OPENED` / `COVER_STATE_CLOSED` |
| `StartCharging` / `StopCharging` | `AssetTelemetryDetails.SubAssetCharging` | becomes `true` / `false` |
| `EnterManualControl` / `ExitManualControl` | `AssetTelemetryDetails.HasActiveManualControlSession` | becomes `true` / `false` |
| `ReturnToHome` | `SubAssetTelemetryDetails.Mode` | moves through `RETURN_AUTO` → `LANDING_AUTO` → back to `IDLE` once docked |

Worked example, using `TakeOff`:

```go
import (
    assetpb "github.com/Zequent/zqnt-client-sdk-go/gen/common/asset/proto"
    livedatapb "github.com/Zequent/zqnt-client-sdk-go/gen/livedata/proto"
)

ld := livedata.New(conn)

stream, err := ld.StreamTelemetry(ctx, "ZQT-DOCK-0417", 1000, 0)
if err != nil {
    log.Fatal(err)
}

for {
    resp, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        // transient error — redial StreamTelemetry yourself here
        break
    }

    data, ok := resp.Telemetry.(*livedatapb.LiveDataTelemetryResponse_Data)
    if !ok {
        continue
    }
    if sub := data.Data.GetSubAsset(); sub != nil && sub.GetMode() == assetpb.SubAssetMode_SUBASSET_MODE_TAKEOFF_FINISHED {
        log.Println("Drone is airborne.")
    }
}
```

Go keeps the full proto enum names as-is (e.g. `SubAssetMode_SUBASSET_MODE_TAKEOFF_FINISHED`), just as a typed enum rather than a string like Python uses.

**What a telemetry frame actually looks like** — one `LiveDataTelemetryResponse` from the drone mid-climb (only the populated fields shown):

```
tid:"f47ac10b-58cc-4372-a567-0e02b2c3d479"  timestamp:{seconds:1756233727}  has_errors:false  sn:"ZQT-DOCK-0417"  asset_id:"550e8400-e29b-41d4-a716-446655440000"
data:{
    sub_asset:{id:"ZQT-DRONE-1123"  latitude:41.015137  longitude:28.97953  absolute_altitude:42.6  relative_altitude:12.4  vertical_speed:1.1  heading:187.5  mode:SUBASSET_MODE_TAKEOFF_AUTO  battery_information:{percentage:"78"  remaining_time:1320  return_to_home_power:"22"}}
}
```

Every frame carries exactly one of `asset` or `sub_asset` via `Telemetry.Source` — never both.

**`asset` does not mean "the dock", and `sub_asset` does not mean "the drone".** They identify
*which entity the reading is about*:

| Populated | The reading describes |
| --- | --- |
| `asset` | the registered top-level Asset itself — a drone, dock, vehicle, sensor gateway or camera |
| `sub_asset` | a child SubAsset belonging to that Asset |

The frame above is a **hierarchical** Dock → Drone configuration. A drone operated independently is
registered as an Asset in its own right, and its readings arrive in `asset` instead — with
`sub_asset` unset:

```
tid:"8f14e45f-ceea-467a-9575-9f2a1b0c3d4e"  timestamp:{seconds:1756233717}  has_errors:false  sn:"SIM-DRONE-001"  asset_id:"c1d7f3a2-95b4-4c1e-8f6d-2a7b9e0c4513"
data:{
    id:"SIM-DRONE-001"  sn:"SIM-DRONE-001"  latitude:52.52000045776367  longitude:13.404999732971191  absolute_altitude:34  relative_altitude:34  wind_speed:3.305775  heading:0
    asset:{mode:ASSET_MODE_IDLE  sub_asset_percentage:100  has_active_manual_control_session:false  position_valid:true  position_state:{gps_number:14  rtk_number:0  quality:5}  network_information:{type:NETWORK_TYPE_4_G  rate:12.4  quality:NETWORK_STATE_QUALITY_GOOD}  manual_control_state:MANUAL_CONTROL_STATE_DISCONNECTED  environment_temp:21.5  humidity:48  rainfall:RAINFALL_NO}
}
```

`GetSubAsset()` returning `nil` here is correct and complete, not missing data — dock-only fields
such as `cover_state` and `air_conditioner` are simply unset for a device that has no enclosure.
Switch on which of `GetAsset()` / `GetSubAsset()` is non-nil rather than assuming a device
category. See [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) for the full model.

See [CONNECTOR_GO.md](CONNECTOR_GO.md) for looking up asset state on demand instead of streaming it, and [QUICKSTART_GO.md](QUICKSTART_GO.md) for how `remotecontrol.New()` / `livedata.New()` get wired up in the first place.
