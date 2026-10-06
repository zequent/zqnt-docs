# Zequent Client SDK (Python) — Functional Responses

A Remote Control call answers once the platform has checked your command and the asset's edge
adapter has replied — but "success" in that answer is easy to misread as "the device did the
thing." This page explains what a command response represents, and how to confirm what really
happened on the device.

For Java, see [FUNCTIONAL_RESPONSES.md](FUNCTIONAL_RESPONSES.md); for Go,
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
3. **Your call answers** once the adapter has replied. The `RemoteControlResponse` you get back *is* that answer;
   there is no separate acknowledgement on another channel.

## What the response tells you

| Response | Meaning |
| --- | --- |
| `success` is `False` | The command was refused before it reached the device (no-fly zone without a detour, a failed pre-flight check), or the adapter rejected it. `error.error_message` says why |
| `success` is `True`, `progress.state == "SKILL_EXECUTION_STATUS_SUCCEEDED"` | The adapter carried out a command that completes at once — dock, charging, reboot and similar commands |
| `success` is `True`, `progress.state == "SKILL_EXECUTION_STATUS_RUNNING"` | The command is under way. `takeoff`, `go_to`, `look_at` and `return_to_home` stay running until the adapter reports their outcome; a command waiting for an approval is running too |

For a running command the outcome arrives later. Follow it with
`client.mission_autonomy.list_skill_executions(asset_sn=...)` (see
[Tracking and controlling an execution](../concepts/applications-and-skills.md#tracking-and-controlling-an-execution)),
or watch telemetry as shown below.

Even `SUCCEEDED` means the adapter reported the command as done — whether that is "the flight
controller accepted it" or "the cover is physically open" depends on the adapter. Telemetry is the
source of truth for device state.

```python
response = await client.remote_control.takeoff(request)

if not response.success:
    print(f"Refused: {response.error.error_message}")
else:
    # e.g. SKILL_EXECUTION_STATUS_RUNNING: the drone is taking off, not airborne yet
    print(response.progress.state)
```

**What that response looks like**, e.g. `print(response)`:

```
RemoteControlResponse(success=True, tid='a1b2c3d4-1234-5678-9abc-def012345678', sn='ZQT-DOCK-0417', asset_id=None, message=None, error=None, progress=ProgressInfo(progress=0.0, state='SKILL_EXECUTION_STATUS_RUNNING', left_time_in_seconds=0.0))
```

A refused command carries `error` instead of `progress`:

```
RemoteControlResponse(success=False, tid='a1b2c3d4-1234-5678-9abc-def012345678', sn='ZQT-DOCK-0417', asset_id=None, message=None, error=ErrorInfo(error_code='ERROR_CODE_CLIENT', error_message="Refused: The destination lies inside no-fly zone 'Airport'.", timestamp=datetime.datetime(2026, 8, 26, 18, 41, 52)), progress=None)
```

## Confirming what actually happened, via telemetry

The response to a command call isn't the source of truth for device state — the telemetry stream is. Subscribe with `client.live_data.stream_telemetry(...)` and watch the relevant field, rather than assuming the command response means the job is done:

| Command(s) | Field to watch | Confirms it worked |
| --- | --- | --- |
| `takeoff()` | `sub_asset_telemetry.mode` | reaches `SUBASSET_MODE_TAKEOFF_FINISHED` (after `TAKEOFF_PREPARE` → `TAKEOFF_AUTO`) |
| `open_cover()` / `close_cover()` | `asset_telemetry.cover_state` | becomes `COVER_STATE_OPENED` / `COVER_STATE_CLOSED` |
| `start_charging()` / `stop_charging()` | `asset_telemetry.sub_asset_charging` | becomes `True` / `False` |
| `enter_manual_control()` / `exit_manual_control()` | `asset_telemetry.has_active_manual_control_session` | becomes `True` / `False` |
| `return_to_home()` | `sub_asset_telemetry.mode` | moves through `RETURN_AUTO` → `LANDING_AUTO` → back to `IDLE` once docked |

Worked example, using `takeoff()`:

```python
response = await client.remote_control.takeoff(request)

if not response.success:
    print(f"Takeoff refused: {response.error.error_message}")
else:
    # The takeoff is running — now watch telemetry for what actually happens.
    async def on_telemetry(telemetry: StreamTelemetryResponse) -> None:
        sub_asset = telemetry.sub_asset_telemetry
        if sub_asset and sub_asset.mode == "SUBASSET_MODE_TAKEOFF_FINISHED":
            print("Drone is airborne.")

    handle = client.live_data.stream_telemetry(
        StreamTelemetryRequest(sn=request.sn),
        on_telemetry,
    )
```

`stream_telemetry` auto-reconnects on transient gRPC errors, so `handle` stays valid across brief disconnects — call `handle.stop()` when you're done watching.

**What a telemetry frame actually looks like** — one `StreamTelemetryResponse` from the drone mid-takeoff (only the populated fields are shown; `SubAssetTelemetry` has more, e.g. `wind_speed`, `height_limit`, `payload`):

```
StreamTelemetryResponse(
    tid='f47ac10b-58cc-4372-a567-0e02b2c3d479',
    sn='ZQT-DOCK-0417',
    timestamp=datetime.datetime(2026, 8, 26, 18, 42, 7, 912000),
    has_errors=False,
    asset_id='550e8400-e29b-41d4-a716-446655440000',
    asset_telemetry=None,
    sub_asset_telemetry=SubAssetTelemetry(
        id='ZQT-DRONE-1123',
        latitude=41.015137,
        longitude=28.97953,
        absolute_altitude=42.6,
        relative_altitude=12.4,
        vertical_speed=1.1,
        heading=187.5,
        mode='SUBASSET_MODE_TAKEOFF_AUTO',
        battery=SubAssetBatteryInfo(percentage='78', remaining_time=1320, return_to_home_power='22')
    ),
    error=None,
)
```

Every frame carries exactly one of `asset_telemetry` or `sub_asset_telemetry` — never both. Note
`mode` is a plain `str` here (the raw proto enum name), unlike Java where it's a typed
`SubAssetMode` enum.

**`asset_telemetry` does not mean "the dock", and `sub_asset_telemetry` does not mean "the drone".**
They identify *which entity the reading is about*:

| Populated field | The reading describes |
| --- | --- |
| `asset_telemetry` | the registered top-level Asset itself — which may be a drone, dock, vehicle, sensor gateway or camera |
| `sub_asset_telemetry` | a child SubAsset belonging to that Asset |

The frame above is a **hierarchical** Dock → Drone configuration: the Asset is a dock, and the drone
is its SubAsset. A drone operated independently is registered as an Asset in its own right, and its
telemetry arrives in `asset_telemetry` instead:

```
StreamTelemetryResponse(
    tid='8f14e45f-ceea-467a-9575-9f2a1b0c3d4e',
    sn='SIM-DRONE-001',
    timestamp=datetime.datetime(2026, 8, 26, 18, 41, 57, 385000),
    has_errors=False,
    asset_id='c1d7f3a2-95b4-4c1e-8f6d-2a7b9e0c4513',
    asset_telemetry=AssetTelemetry(
        id='SIM-DRONE-001',
        sn='SIM-DRONE-001',
        latitude=52.52000045776367,
        longitude=13.404999732971191,
        absolute_altitude=34.0,
        relative_altitude=34.0,
        wind_speed=3.305775,
        heading=0.0,
        mode='ASSET_MODE_IDLE',
        sub_asset_percentage=100.0,
        has_active_manual_control_session=False,
        position_valid=True,
        position_state=AssetPositionState(gps_number=14, rtk_number=0, quality=5),
        network_information=AssetNetworkInfo(type='NETWORK_TYPE_4_G', rate=12.4, quality='NETWORK_STATE_QUALITY_GOOD'),
        manual_control_state='MANUAL_CONTROL_STATE_DISCONNECTED',
        environment_temp=21.5,
        humidity=48.0,
        rainfall='RAINFALL_NO',
        cover_state=None,        # dock enclosure hardware — a standalone drone has none
        air_conditioner=None,
        inside_temp=None,
    ),
    sub_asset_telemetry=None,
    error=None,
)
```

`sub_asset_telemetry=None` here is correct and complete, not missing data. `AssetTelemetry` is a
superset covering every kind of Asset; each device populates the fields that physically apply to
it. See [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) for the full model.

Note `SubAssetBatteryInfo.percentage` and `return_to_home_power` are `str`, not numbers.

See [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md) for looking up asset state on demand instead of streaming it, and [QUICKSTART_PYTHON.md](QUICKSTART_PYTHON.md) for how `client.remote_control` / `client.live_data` get wired up in the first place.
