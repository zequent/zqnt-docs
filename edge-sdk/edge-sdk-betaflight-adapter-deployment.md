# Betaflight Edge Adapter -- Deployment Guide

The Betaflight Edge Adapter connects a Betaflight-based flight controller (FC) to the Zequent platform over a direct serial/USB connection, arming and controlling it via RC-style channel commands.

**Status: source-only, very early.** No `Dockerfile` or CI pipeline exists yet for this adapter — it's run directly from source. Because it needs a serial device (`/dev/ttyACM0` or similar) passed through, containerizing it will need `--device`/privileged access whenever that packaging does land.

---

## Prerequisites

- Python 3.12+ and [`uv`](https://docs.astral.sh/uv/)
- A Betaflight flight controller reachable over serial/USB from the machine running the adapter
- An AUX channel on the FC's Betaflight modes configured as the arm switch
- Running Zequent platform services: Connector Service, Live Data Service

---

## Implemented Commands

| Command | Notes |
| --- | --- |
| `RegisterAsset` / `DeRegisterAsset` | Registers/removes the FC as an asset with the Connector Service |
| `GetCapabilities` | Reports supported commands |
| `EnterManualControl` / `ExitManualControl` | Opens/closes the RC-override session with the FC |
| `TakeOff` | Open-loop throttle ramp to a configured hover value — **not** altitude-hold; there is no barometer/GPS feedback loop |
| `ManualControlInput` | Streams joystick input to RC channel overrides |

Everything else defaults to "not supported."

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `BETAFLIGHT_CONNECTION` | `/dev/ttyACM0` | Serial port of the FC |
| `BETAFLIGHT_PRECONNECT_SN` | _unset_ | Serial number to connect to immediately on startup. Not just a local-testing convenience — leaving it unset logs a warning and the board only connects once the platform pushes an asset registration, so set it for a normal single-board deployment too |
| `BETAFLIGHT_ARM_CHANNEL` | `5` | AUX channel (5-8 = AUX1-AUX4) the arm switch is bound to — must match your Betaflight modes configuration |
| `BETAFLIGHT_ARM_VALUE` | `1800` | PWM µs value (1000-2000) sent on the arm channel to arm |
| `BETAFLIGHT_DISARM_VALUE` | `1000` | PWM µs value sent to disarm |
| `BETAFLIGHT_TAKEOFF_HOVER` | `1500` | Target throttle PWM (µs) at the end of the takeoff ramp — tune to your quad |
| `BETAFLIGHT_TAKEOFF_RAMP_S` | `3.0` | Duration of the takeoff throttle ramp, in seconds |
| `GRPC_HOST` | `0.0.0.0` | gRPC server bind host |
| `GRPC_PORT` | `50051` | gRPC server bind port |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `50053` | Connector Service port — override to `8010` for a real deployment |
| `TELEMETRY_HOST` | `localhost` | Live Data Service host |
| `TELEMETRY_PORT` | `50052` | Live Data Service port — override to `8003` for a real deployment |
| `ADAPTER_SN` | _empty_ | Default serial number attached to outgoing telemetry frames, and logged at startup |
| `REDIS_URL` | _unset_ | Optional — Redis for vendor/asset caching |
| `EDGE_ENDPOINT` | _unset_ | Optional service-discovery registration endpoint — also requires `ASSET_TYPE` and `ASSET_VENDOR` to actually register; missing either silently skips registration (logged as a warning only) |
| `ASSET_TYPE` | _unset_ | Full proto-style name, **not** the bare enum member — `ASSET_TYPE_AIRCRAFT`, not `AIRCRAFT` |
| `ASSET_VENDOR` | _unset_ | Same requirement — `ASSET_VENDOR_BETAFLIGHT`, not `BETAFLIGHT` |
| `LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `LOG_FORMAT` | `json` | `json` or `text` |

The library's own built-in `CONNECTOR_PORT`/`TELEMETRY_PORT` defaults (`50053`/`50052`) do not match the platform's real service ports (`8010`/`8003`) — set them explicitly for your deployment.

---

## Running from source

```bash
uv sync
ls /dev/ttyACM* /dev/ttyUSB*   # find your FC's serial port
cp .env.example .env           # then edit BETAFLIGHT_CONNECTION and the rest
uv run --env-file .env edge-betaflight
```

---

## Safety notes

- **Verify `BETAFLIGHT_ARM_CHANNEL`/`BETAFLIGHT_ARM_VALUE` match your FC's actual Betaflight modes configuration before running.** A mismatch can leave the arm switch unresponsive, or worse, arm unexpectedly.
- `TakeOff` is an open-loop ramp — it does not hold altitude. Treat it as "spin up to a rough hover throttle," not an autonomous takeoff. Confirmed in source: after the ramp, the FC stays armed at hover throttle and the caller **must** immediately follow up with `ManualControlInput` — if nothing takes over, Betaflight's own RC failsafe will trigger.
- Test with props off first when validating a new configuration.

---

## Port

The adapter listens on `50051` for incoming gRPC commands from the platform.
