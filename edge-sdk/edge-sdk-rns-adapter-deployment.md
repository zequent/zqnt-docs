# RNS Edge Adapter -- Deployment Guide

The RNS Edge Adapter bridges devices reachable over [Reticulum (RNS)](https://reticulum.network/) mesh networking to the Zequent platform. It's an early-stage adapter: it implements asset registration and vendor-specific commands through the standard custom-command mechanism, but does not yet implement the standard flight/dock command set (`TakeOff`, `GoTo`, etc.) that the DJI or MAVLink adapters do.

**Incoming telemetry/detection publishing is not implemented yet, independent of any environment
variable.** Confirmed in source: `RnsProtocol._handle_packet` (the handler for every packet received
from the RNS master node) currently only logs the decoded payload — it's an explicit `# TODO: parse
message ... and publish to ZQNT platform`, with no call to a telemetry or detection publisher
anywhere in the adapter. Setting `TELEMETRY_HOST`/`TELEMETRY_PORT` wires up the connection but has
nothing to actually publish yet.

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
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `8010` | Connector Service port |
| `TELEMETRY_HOST` | _unset_ (empty string) | Live Data Service host — connection setup only; see the note above, nothing is published yet regardless of this value |
| `TELEMETRY_PORT` | `50052` | Live Data Service port — override to `8003` for a real deployment |
| `RNS_MASTER_DEST_HASH` | _unset_ | Destination hash of the RNS master node this adapter talks to |
| `RNS_CONFIG_PATH` | _unset_ | Path to a Reticulum config directory, if not using the default `~/.reticulum` |
| `EDGE_ENDPOINT` | _unset_ | Endpoint advertised via optional Redis service-discovery registration — also requires `ASSET_TYPE` and `ASSET_VENDOR` to actually register; missing either silently skips registration (logged as a warning only) |
| `ASSET_TYPE` | _unset_ | Full proto-style name, **not** the bare enum member — `ASSET_TYPE_AIRCRAFT`, not `AIRCRAFT` |
| `ASSET_VENDOR` | _unset_ | Same requirement — `ASSET_VENDOR_RNS`, not `RNS` |
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
