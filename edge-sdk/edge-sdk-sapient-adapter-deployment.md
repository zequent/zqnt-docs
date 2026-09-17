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
| `PrepareTask` / `StartTask` / `StopTask` | Task lifecycle, keyed by `task_id` |
| `GoTo` | Sent to the node as a SAPIENT task |
| `LookAt` | Sent to the node as a SAPIENT task |
| `EnableGimbalTracking` | Sent to the node as a SAPIENT control message |
| Custom commands | Vendor-specific SAPIENT messages not covered above go through `SendCustomCommand` |

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `SAPIENT_HOST` | `0.0.0.0` | Bind address for the TCP SAPIENT listener |
| `SAPIENT_PORT` | `14000` | TCP port SAPIENT edge nodes connect to |
| `GRPC_PORT` | `50051` | gRPC port the platform reaches this adapter on |
| `ZQNT_CLAIM_CODE` | _unset_ | A one-time pairing code from the console's **Pair device** dialog. Used only when the platform does not already know a serial number this adapter is bringing up: the code decides which organization the resulting asset belongs to, and that cannot be changed afterwards. Leave it unset once the assets exist — an already-paired device does not need it, and the code is single-use. Without it, an unknown serial simply has no asset, and the adapter creates nothing. |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `8010` | Connector Service port |
| `TELEMETRY_HOST` | `localhost` | Live Data Service host |
| `TELEMETRY_PORT` | `8003` | Live Data Service port |
| `REDIS_URL` | _unset_ | Optional — enables Redis-backed caching when set |

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
