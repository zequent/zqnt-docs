# Edge SDK (Python) — Quickstart Guide

This guide walks you through creating a new edge adapter project from scratch using the Zequent Edge SDK for Python. By the end, you will have a running Python application that receives commands from the platform and pushes telemetry data.

## Prerequisites

- Python **3.12+**
- [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`
- Docker or Podman (for running platform services)
- GitHub account with access to the Zequent packages repository

---

## Step 1: Create a new project

```bash
mkdir my-edge-adapter && cd my-edge-adapter
uv init --python 3.12
```

This creates a `pyproject.toml` and a `.venv/` managed by `uv`.

---

## Step 2: Add the Edge SDK dependency

The SDK is not on PyPI yet. Install it from its Git release tag, together with `zqnt-utils`, the
Zequent package it depends on (also from Git). In `pyproject.toml`:

```toml
[project]
dependencies = [
    "edge-python-sdk>=2.0.0",
]

[tool.uv.sources]
edge-python-sdk = { git = "https://github.com/zequent/zqnt-edge-sdk-python", tag = "v2.0.0" }
zqnt-utils = { git = "https://github.com/zequent/zqnt-utils-python", tag = "v2.0.0" }
```

then `uv sync` — both repositories are private, so you need GitHub access to them. With pip:

```bash
pip install "zqnt-utils @ git+https://github.com/zequent/zqnt-utils-python@v2.0.0" \
            "edge-python-sdk @ git+https://github.com/zequent/zqnt-edge-sdk-python@v2.0.0"
```

---

## Step 3: Configure the adapter

`EdgeAdapterConfig` centralizes all environment-variable reading for you — you normally don't construct it by hand, just call `EdgeAdapterConfig.from_env()`. Create a `.env` file (or export environment variables in your shell):

```bash
# gRPC server this adapter exposes to the platform
GRPC_HOST=0.0.0.0
GRPC_PORT=9001

# Platform services this adapter talks to — override these to match your deployment
CONNECTOR_HOST=localhost
CONNECTOR_PORT=8010
TELEMETRY_HOST=localhost
TELEMETRY_PORT=8003
MISSION_AUTONOMY_HOST=localhost
MISSION_AUTONOMY_PORT=8004

ADAPTER_SN=YOUR_DEVICE_SERIAL_NUMBER
LOG_LEVEL=INFO

# Edge credential: the Admin Console, Manage -> Access & Integrations -> Credentials (kind "Edge adapter")
ZQNT_EDGE_TOKEN=<your edge credential>
# The platform's public key, to check the platform's calls into the adapter
ZQNT_PLATFORM_PUBLIC_KEY=<the platform's service public key>
# Optional: a one-time pairing code, for a device the platform doesn't know yet
# ZQNT_CLAIM_CODE=<pairing code>
```

Every call the runtime makes to the platform carries `ZQNT_EDGE_TOKEN`; without it the platform
refuses them. The adapter's server refuses every call that the platform did not sign with the key in
`ZQNT_PLATFORM_PUBLIC_KEY` — `ZQNT_EDGE_AUTH_DISABLED=true` turns that off, for a local stack only.

The library's own built-in defaults for `CONNECTOR_PORT`/`TELEMETRY_PORT`/`MISSION_AUTONOMY_PORT` (`50053`/`50052`/`50054`) do **not** match the platform's real service ports (`8010`/`8003`/`8004`) — always set these explicitly for your deployment rather than relying on the defaults.

See [Configuration reference](edge-sdk-python-configuration.md) for the complete list, including optional Redis-based service-discovery registration.

---

## Step 4: Implement the `EdgeAdapter`

Create `my_edge_adapter/adapter.py`. You only override the commands your hardware supports; everything else returns `EdgeResponse.not_supported(...)` automatically.

```python
import logging
from edge_sdk import (
    EdgeAdapter, EdgeResponse, ErrorMessage, ErrorCode,
    Capabilities, AssetType,
    RequestContext, Coordinates,
    ReturnToHomeRequest,
)

log = logging.getLogger(__name__)


class MyDeviceAdapter(EdgeAdapter):

    async def get_capabilities(self, sn: str, asset_id: str | None) -> Capabilities:
        # Auto-discovers which commands you implemented.
        return self._auto_capabilities(sn, AssetType.DOCK)

    async def take_off(self, ctx: RequestContext, coordinates: Coordinates) -> EdgeResponse:
        log.info("Takeoff requested for SN: %s", ctx.sn)
        # call your hardware's takeoff API here
        success = True
        if success:
            return EdgeResponse.ok(ctx.tid, ctx.sn, "Takeoff initiated")
        return EdgeResponse.fail(ctx.tid, ctx.sn,
            ErrorMessage(message="Takeoff failed", code=ErrorCode.ASSET_ERROR))

    async def return_to_home(
        self, ctx: RequestContext, request: ReturnToHomeRequest
    ) -> EdgeResponse:
        log.info("RTH for SN: %s", ctx.sn)
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Returning to home")

    async def open_cover(self, ctx: RequestContext) -> EdgeResponse:
        log.info("Opening cover for SN: %s", ctx.sn)
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Cover opening")
```

---

## Step 5: Wire it together and run

`EdgeAdapterConfig.runtime()` gives you an async context manager that connects `ConnectorClient`, `TelemetryPublisher`, and `MissionAutonomyClient` for you, then `runtime.serve(adapter)` starts the gRPC server (with the standard health-check service pre-registered) and blocks until termination:

```python
# my_edge_adapter/__main__.py
import asyncio
import logging

from edge_sdk import EdgeAdapterConfig
from .adapter import MyDeviceAdapter


async def main():
    config = EdgeAdapterConfig.from_env()
    async with config.runtime() as runtime:
        adapter = MyDeviceAdapter()
        await runtime.serve(adapter)


if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
uv run python -m my_edge_adapter
```

You should see a startup log line reporting the gRPC/Connector/Telemetry/MissionAutonomy addresses it connected with.

If your adapter needs to look up or pair its own asset on startup, use `runtime.connector` (a connected `ConnectorClient`) before calling `runtime.serve(...)` — see [Connector](edge-sdk-python-connector.md#typical-startup-pattern). If it needs to push telemetry, pass `runtime.telemetry` (a connected `TelemetryPublisher`) into your adapter's constructor — see [Live Data](edge-sdk-python-live-data.md).

---

## Step 6: Verify against the platform

With the platform services running from the published container images (see [Zequent Documentation](../README.md)):

1. The platform discovers your adapter via the configured endpoint.
2. Sending a takeoff command from a Client SDK (Java or Python) lands on your `take_off` method.
3. Telemetry pushed via `runtime.telemetry` is visible through the Live Data service.

To call it by hand with `grpcurl`, run it with `ZQNT_EDGE_AUTH_DISABLED=true` — locally only. The
server has no gRPC reflection, so point `grpcurl` at the protocol files (`edge.proto` from the
`zqnt-protos` repository):

```bash
grpcurl -plaintext -import-path zqnt-protos -proto edge.proto -d '{
  "base": {"sn": "YOUR_DEVICE_SN", "tid": "test-123", "timestamp": "2026-01-01T00:00:00Z"},
  "coordinate": {"latitude": 47.3769, "longitude": 8.5417, "altitude": 100.0}
}' localhost:9001 zqnt.EdgeAdapterService/TakeOff
```

---

## Next steps

- [Configuration reference](edge-sdk-python-configuration.md) — every env var the SDK reads
- [Adapter reference](edge-sdk-python-adapter.md) — full list of overridable methods
- [Live Data reference](edge-sdk-python-live-data.md) — telemetry, detections, and notifications
- [Connector reference](edge-sdk-python-connector.md) — asset pairing and the Skill Registry
