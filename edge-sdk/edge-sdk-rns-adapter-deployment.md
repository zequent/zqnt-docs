# RNS Edge Adapter -- Deployment Guide

The RNS Edge Adapter bridges devices reachable over [Reticulum (RNS)](https://reticulum.network/) mesh networking to the Zequent platform. It's an early-stage adapter: it implements asset registration and vendor-specific commands through the standard custom-command mechanism, but does not yet implement the standard flight/dock command set (`TakeOff`, `GoTo`, etc.) that the DJI or MAVLink adapters do.

**Status: source-only.** Real, working code with its own `Dockerfile` and CI pipeline, but no version has ever been tagged, so no image has been published yet. Run it from source with `uv`, or build your own image.

---

## Prerequisites

- Python 3.12+ and [`uv`](https://docs.astral.sh/uv/)
- A Reticulum network reachable from this adapter, and the destination hash of the RNS master node it should talk to
- Running Zequent platform services: Connector Service, Live Data Service

---

## Implemented Commands

| Command | Notes |
| --- | --- |
| `RegisterAsset` | Registers the RNS-connected device as an asset with the Connector Service |
| `GetCapabilities` | Reports supported commands |
| `SendCustomCommand` | The primary command path today — vendor-specific RNS messages are dispatched through this rather than dedicated flight/dock methods |

Everything else defaults to "not supported." If you need the standard command set for an RNS device, implement it in your own fork by overriding the relevant `EdgeAdapter` methods — see [Edge Adapter](edge-sdk-python-adapter.md).

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `GRPC_HOST` | `[::]` | gRPC server bind host |
| `GRPC_PORT` | `50051` | gRPC server bind port |
| `ZQNT_CLAIM_CODE` | _unset_ | A one-time pairing code from the console's **Pair device** dialog. Used only when the platform does not already know a serial number this adapter is bringing up: the code decides which organization the resulting asset belongs to, and that cannot be changed afterwards. Leave it unset once the assets exist — an already-paired device does not need it, and the code is single-use. Without it, an unknown serial simply has no asset, and the adapter creates nothing. |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `8010` | Connector Service port |
| `TELEMETRY_HOST` | _unset_ | Live Data Service host — telemetry forwarding is disabled while this is unset |
| `TELEMETRY_PORT` | `50052` | Live Data Service port — override to `8003` for a real deployment |
| `RNS_MASTER_DEST_HASH` | _unset_ | Destination hash of the RNS master node this adapter talks to |
| `RNS_CONFIG_PATH` | _unset_ | Path to a Reticulum config directory, if not using the default `~/.reticulum` |
| `EDGE_ENDPOINT` | _unset_ | Endpoint advertised via optional Redis service-discovery registration |
| `REDIS_URL` | `redis://localhost:6379` | Used only when `EDGE_ENDPOINT` is set |

---

## Running from source

```bash
uv sync
uv run edge-rns
```

`.env`:

```bash
GRPC_PORT=50051
CONNECTOR_HOST=localhost
CONNECTOR_PORT=8010
TELEMETRY_HOST=localhost
TELEMETRY_PORT=8003
RNS_MASTER_DEST_HASH=<your RNS destination hash>
```

---

## Building your own image

```bash
cd rns-adapter
docker build -t your-registry/edge-rns:local .
docker run --env-file .env -p 50051:50051 your-registry/edge-rns:local
```

---

## Port

The adapter listens on `50051` for incoming gRPC commands from the platform.
