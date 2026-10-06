# Betaflight Edge Adapter — Deployment Guide

The Betaflight Edge Adapter connects a Betaflight-based flight controller (FC) to the Zequent platform over a direct serial/USB connection, arming and controlling it via RC-style channel commands.

The published image is `ghcr.io/zequent/zqnt-adapter-betaflight:2.0.0`; you can also run it from source. It needs the flight controller's serial device (`/dev/ttyACM0` or similar), so the container needs it passed through with `--device`.

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
| `ZQNT_CLAIM_CODE` | _unset_ | A one-time pairing code from the console's **Pair device** dialog. Used only when the platform does not already know a serial number this adapter is bringing up: the code decides which organization the resulting asset belongs to, and that cannot be changed afterwards. Leave it unset once the assets exist — an already-paired device does not need it, and the code is single-use. Without it, an unknown serial simply has no asset, and the adapter creates nothing. |
| `ZQNT_EDGE_TOKEN` | _unset_ | The adapter's **edge credential**, sent on every call to the platform. Issued in the Admin Console under **Manage → Access & Integrations → Credentials** (kind *Edge adapter*); without it the platform refuses the calls |
| `ZQNT_PLATFORM_PUBLIC_KEY` | _unset_ | The platform's public key (alias `SERVICE_AUTH_PUBLIC_KEY`). The adapter refuses every command the platform did not sign with it; without it, every command is refused |
| `ZQNT_EDGE_AUTH_DISABLED` | `false` | `true` accepts commands from anyone who can reach the port — local stacks only |
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

## Running the published image

The adapter needs the flight controller's serial device passed into the container:

```bash
docker run --env-file .env --device /dev/ttyACM0 -p 50051:50051 ghcr.io/zequent/zqnt-adapter-betaflight:2.0.0
```

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
