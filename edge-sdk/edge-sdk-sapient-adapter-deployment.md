# Sapient Edge Adapter -- Deployment Guide

The Sapient Edge Adapter bridges TCP SAPIENT-protocol edge nodes to the Zequent platform's gRPC command interface. Incoming SAPIENT messages are translated to the platform's telemetry format; outgoing platform commands are converted to SAPIENT tasks and sent back over TCP.

**Status: source-only.** This adapter has real, working code and its own `Dockerfile`, but no version has been tagged yet, so there is no published `ghcr.io/zequent/...` image to pull today. Run it from source with `uv`, or build your own image from the adapter's `Dockerfile`.

---

## Prerequisites

- Python 3.12+ and [`uv`](https://docs.astral.sh/uv/)
- A SAPIENT-compatible edge node that can open a TCP connection to this adapter
- Running Zequent platform services: Connector Service, Live Data Service

---

## Implemented Commands

| Command | Notes |
| --- | --- |
| `RegisterAsset` | Registers the SAPIENT node as an asset with the Connector Service |
| `GetCapabilities` | Reports supported commands |
| `PrepareTask` | No-op acknowledgment — SAPIENT has no explicit prepare step |
| `StartTask` / `StopTask` | Sent as a SAPIENT `CONTROL_START`/`CONTROL_STOP` control command, keyed by `task_id` |
| `GoTo` | Sent to the node as a SAPIENT task with a `move_to` command |
| `LookAt` | Sent to the node as a SAPIENT task with a `look_at` command |
| `EnableGimbalTracking` | Also sent as a SAPIENT task, not a control command — repurposed as SAPIENT's detection-threshold setting (`enabled` maps to `DISCRETE_THRESHOLD_HIGH`/`LOW`), not literal gimbal tracking |
| Custom commands | Vendor-specific SAPIENT messages not covered above go through `SendCustomCommand` |

---

## Environment Variables

SAPIENT-specific variables are read directly by this adapter's own `main.py`. Everything else comes
from the shared Python Edge SDK's `EdgeAdapterConfig.from_env()`, which this adapter passes no
overrides to — confirmed against `edge_sdk/config.py`, these are that class's own real defaults, not
Sapient-specific values:

| Variable | Default | Purpose |
| --- | --- | --- |
| `SAPIENT_HOST` | `0.0.0.0` | Bind address for the TCP SAPIENT listener |
| `SAPIENT_PORT` | `14000` | TCP port SAPIENT edge nodes connect to |
| `GRPC_HOST` | `0.0.0.0` | gRPC server bind host |
| `GRPC_PORT` | `50051` | gRPC port the platform reaches this adapter on |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `50053` | **Not** the real Connector Service platform port (`8010`, see the [image table](../README.md#platform-service-images)); set it explicitly, don't rely on this default |
| `TELEMETRY_HOST` | `localhost` | Live Data Service host |
| `TELEMETRY_PORT` | `50052` | Likewise not the real platform port (`8003`); set it explicitly |
| `ADAPTER_SN` | _empty_ | Serial number used on outgoing telemetry frames |
| `EDGE_ENDPOINT` | _unset_ | Endpoint this adapter advertises via optional Redis service-discovery registration |
| `ASSET_TYPE` | _unset_ | Full proto-style name, **not** the bare enum member — `ASSET_TYPE_AIRCRAFT`, not `AIRCRAFT`. A bare value fails `proto_enum_lookup` and silently skips registration (logged as a warning only) |
| `ASSET_VENDOR` | _unset_ | Same requirement — `ASSET_VENDOR_SAPIENT`, not `SAPIENT` |
| `LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `LOG_FORMAT` | `json` | `json` or `text` |
| `REDIS_URL` | _unset_ (checked directly, not via the shared config's own default) | Enables this adapter's own node online/offline caching when set — independent of `EDGE_ENDPOINT`'s separate, optional use of Redis for service-discovery registration |

---

## Running from source

```bash
uv sync --all-extras
uv run --env-file .env edge-sapient
```

`.env`:

```bash
SAPIENT_HOST=0.0.0.0
SAPIENT_PORT=14000
GRPC_PORT=50051
CONNECTOR_HOST=localhost
CONNECTOR_PORT=8010
TELEMETRY_HOST=localhost
TELEMETRY_PORT=8003
ADAPTER_SN=sapient-01
ASSET_TYPE=ASSET_TYPE_AIRCRAFT
ASSET_VENDOR=ASSET_VENDOR_SAPIENT
```

---

## Building your own image

```bash
cd sapient-adapter
docker build -t your-registry/edge-sapient:local .
docker run --env-file .env -p 14000:14000 -p 50051:50051 your-registry/edge-sapient:local
```

---

## Ports

| Port | Purpose |
| --- | --- |
| `14000` | TCP — incoming SAPIENT edge-device connections |
| `50051` | gRPC — outgoing platform command interface |
