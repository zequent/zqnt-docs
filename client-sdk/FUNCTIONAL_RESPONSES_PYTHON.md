# Zequent Client SDK (Python) — Functional Responses

Every RemoteControl call returns a response as soon as the edge adapter replies — but "success" in that response is easy to misread as "the device did the thing." This page explains what a command response actually represents, and how to confirm what really happened on the device.

For Java, see [FUNCTIONAL_RESPONSES.md](FUNCTIONAL_RESPONSES.md).

## Command calls are synchronous, not fire-and-forget

`await client.remote_control.takeoff(request)` doesn't return the moment your call is accepted — it awaits until the device's own command handler has actually responded. There's no separate "ack" that arrives later on some other channel: the `RemoteControlResponse` you get back *is* that answer.

## What `success` actually means

A successful response means the adapter's handler for that command returned without error — nothing more. It's entirely up to the adapter's own implementation whether that means "the flight controller accepted the takeoff command" or "the drone has physically left the ground." The platform doesn't distinguish the two, and neither does the response:

```python
response = await client.remote_control.takeoff(request)

if response.success:
    # The adapter accepted and initiated the command.
    # This does NOT mean the drone is airborne yet.
    print(response.message)  # e.g. "Takeoff initiated"
else:
    print(response.error.error_message)
```

Treat a successful response as "the command was handed off and accepted," not as confirmation of the physical outcome. For that, you need telemetry.

**What that response actually looks like**, e.g. `print(response)`:

```
RemoteControlResponse(success=True, tid='a1b2c3d4-1234-5678-9abc-def012345678', sn='ZQT-DOCK-0417', asset_id='550e8400-e29b-41d4-a716-446655440000', message='Takeoff initiated', error=None, progress=None)
```

A rejected command looks the same shape, with `error` populated instead of `message`:

```
RemoteControlResponse(success=False, tid='a1b2c3d4-1234-5678-9abc-def012345678', sn='ZQT-DOCK-0417', asset_id='550e8400-e29b-41d4-a716-446655440000', message=None, error=ErrorInfo(error_code='ERROR_CODE_ASSET', error_message='Asset ZQT-DOCK-0417 is not connected', timestamp=datetime.datetime(2026, 8, 26, 18, 41, 52)), progress=None)
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
    print(f"Adapter rejected takeoff: {response.error.error_message}")
else:
    # Command was accepted — now watch telemetry for what actually happens.
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
