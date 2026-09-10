# Zequent Edge SDK (Python)

The Edge SDK is used to build custom edge adapters that connect physical assets such as drones, docks, vehicles, and other remote devices to Zequent. A custom adapter exposes a consistent command and telemetry interface to the platform and to Client SDK consumers.

This is the **Python** edition of the Edge SDK. For Java, see [edge-sdk-overview.md](edge-sdk-overview.md).

## Tech Specs

| Requirement   | Version                       |
|---------------|-------------------------------|
| Python        | 3.12+                         |
| Package mgr   | `uv` (recommended) or `pip`   |
| gRPC runtime  | `grpcio` / `grpcio-tools`     |
| Async runtime | `asyncio` (`grpc.aio`)        |

## Overview

The Edge SDK sits between the physical device and the Zequent platform services. An edge adapter is a standalone Python application that depends on the `edge-python-sdk` package and provides concrete implementations for the commands relevant to its hardware.

```
+-------------------+     gRPC     +------------------------+
| Zequent Platform  | <----------> |  Your Edge Adapter     |
| (container images)|              |  (Python application)  |
+-------------------+              |  - EdgeAdapter subclass|
                                   |  - EdgeServer          |
                                   |  - TelemetryPublisher  |
                                   +------------------------+
                                              |
                                              v
                                       Physical hardware
```

## Core Concepts

### `EdgeAdapter`

An `abc.ABC` you subclass to provide hardware-specific behaviour. Only `get_capabilities` is abstract — every other method has a default implementation that returns `EdgeResponse.not_supported(...)`. Override only the commands your hardware supports.

### `EdgeServer`

The async gRPC server that hosts your `EdgeAdapter`. Routes incoming RPC calls to your overridden methods, converts protobuf messages to Python dataclasses, and reports unimplemented methods correctly.

### `TelemetryPublisher`

Manages a persistent gRPC connection to the Live Data service. Lets you push asset and sub-asset telemetry using strongly-typed dataclasses (`AssetTelemetry`, `SubAssetTelemetry`).

### `ConnectorClient`

Talks to the platform's Connector Service over gRPC for asset registration and lookup. Mission/task management is not part of this client; the platform drives task execution by calling into your `EdgeAdapter` instead (see [Mission Autonomy](edge-sdk-python-mission-autonomy.md)).

### `EdgeAdapterConfig` / `EdgeAdapterRuntime`

`EdgeAdapterConfig.from_env()` centralizes all environment-variable reading; its `.runtime()` gives you an async context manager (`EdgeAdapterRuntime`) that connects `ConnectorClient`, `TelemetryPublisher`, and `MissionAutonomyClient` for you and exposes `serve(adapter)` to start the gRPC server. This is the recommended way to wire up `main()` — see the [Quickstart](edge-sdk-python-quickstart.md).

## Available Documentation

| Document                                                                  | Description                                                          |
|---------------------------------------------------------------------------|----------------------------------------------------------------------|
| [Quickstart](edge-sdk-python-quickstart.md)                               | Get a new Python edge adapter project running in minutes             |
| [Configuration](edge-sdk-python-configuration.md)                         | All configuration knobs and environment variable mappings            |
| [Edge Adapter](edge-sdk-python-adapter.md)                                | Implementing the `EdgeAdapter` base class                            |
| [Live Data](edge-sdk-python-live-data.md)                                 | Producing telemetry data streams from your adapter                   |
| [Connector](edge-sdk-python-connector.md)                                 | Asset and resource management via the Connector Service              |
| [Mission Autonomy](edge-sdk-python-mission-autonomy.md)                   | Scheduler lookup, task lifecycle, and custom commands                |
| [Models Reference](edge-sdk-python-models.md)                             | Request, response, and telemetry data model reference                |

Ready-made adapters built on this SDK, and their deployment guides:

| Adapter | Guide |
|---------|-------|
| MAVLink (PX4/ArduPilot) | [Deployment guide](edge-sdk-mavlink-adapter-deployment.md) |
| Sapient | [Deployment guide](edge-sdk-sapient-adapter-deployment.md) |
| RNS (Reticulum mesh) | [Deployment guide](edge-sdk-rns-adapter-deployment.md) |
| Betaflight | [Deployment guide](edge-sdk-betaflight-adapter-deployment.md) |
| AI (YOLO detection) | [Deployment guide](edge-sdk-ai-adapter-deployment.md) |

## Quick Start

Install the Edge SDK:

```bash
uv add edge-python-sdk
# or
pip install edge-python-sdk
```

Configure your edge via environment variables (see [Configuration](edge-sdk-python-configuration.md) for the full list; the platform's real service ports are `8010`/`8003`/`8004`, not the library's own defaults):

```bash
export GRPC_PORT=9001
export CONNECTOR_HOST=localhost
export CONNECTOR_PORT=8010
export TELEMETRY_HOST=localhost
export TELEMETRY_PORT=8003
export MISSION_AUTONOMY_HOST=localhost
export MISSION_AUTONOMY_PORT=8004
export ADAPTER_SN=YOUR_DEVICE_SERIAL_NUMBER
```

Implement the adapter:

```python
import asyncio
from edge_sdk import (
    EdgeAdapter, EdgeAdapterConfig, EdgeResponse,
    Capabilities, AssetType, RequestContext, Coordinates,
)


class MyEdgeAdapter(EdgeAdapter):

    async def get_capabilities(self, sn: str, asset_id: str | None) -> Capabilities:
        return self._auto_capabilities(sn, AssetType.DOCK)

    async def take_off(self, ctx: RequestContext, coordinates: Coordinates) -> EdgeResponse:
        # call your hardware-specific takeoff
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Takeoff initiated")


async def main():
    config = EdgeAdapterConfig.from_env()
    async with config.runtime() as runtime:
        await runtime.serve(MyEdgeAdapter())


asyncio.run(main())
```

For a complete walkthrough, see the [Quickstart Guide](edge-sdk-python-quickstart.md).
