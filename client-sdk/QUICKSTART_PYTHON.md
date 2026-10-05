# Zequent Client SDK (Python) - Quick Start Guide

## For Customers: Using the SDK in Your Project

This guide shows you how to use the Zequent Python Client SDK in your application.

For Java, see [QUICKSTART.md](QUICKSTART.md).

---

## Step 1: Add the dependency

The SDK is not on PyPI yet. Install it from its Git release tag, together with `zqnt-utils`, the
Zequent package it depends on (also from Git). Both repositories are private: you need GitHub
access to them.

`pyproject.toml` (uv-managed projects):

```toml
[project]
dependencies = [
    "zqnt-client-sdk>=2.0.0",
]

[tool.uv.sources]
zqnt-client-sdk = { git = "https://github.com/zequent/zqnt-client-sdk-python", tag = "v2.0.0" }
zqnt-utils = { git = "https://github.com/zequent/zqnt-utils-python", tag = "v2.0.0" }
```

then `uv sync`. With pip:

```bash
pip install "zqnt-utils @ git+https://github.com/zequent/zqnt-utils-python@v2.0.0" \
            "zqnt-client-sdk @ git+https://github.com/zequent/zqnt-client-sdk-python@v2.0.0"
```

That's it for dependencies. Everything is auto-discovered from environment variables.

---

## Step 2: Configuration

Create a `.env` file in your project root (or export the variables in your shell):

```bash
# .env
ZQNT_CLIENT_TOKEN=<your client credential>

REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002

MISSION_AUTONOMY_SERVICE_HOST=localhost
MISSION_AUTONOMY_SERVICE_PORT=8004

LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003

CONNECTOR_SERVICE_HOST=localhost
CONNECTOR_SERVICE_PORT=8010
```

`ZQNT_CLIENT_TOKEN` is the application's **client credential**, issued in the Admin Console under
**Manage → Access & Integrations → Credentials**. Every call carries it; without one the platform
refuses every call. See [Configuration](CONFIGURATION_PYTHON.md#client-credential-zqnt_client_token).

If you're using `python-dotenv`, load it before constructing the client:

```python
from dotenv import load_dotenv
load_dotenv()
```

The Python SDK has **no DI container** — there is no equivalent of CDI / `@Inject`. You instantiate `ZequentClient` once and pass it where it's needed (FastAPI dependency, app singleton, etc.).

---

## Step 3: Use the client

### Option A: as an async context manager (recommended)

```python
import asyncio
from client_sdk import ZequentClient, TakeoffRequest, ReturnToHomeRequest


async def main():
    async with ZequentClient.from_env() as client:
        # Takeoff
        await client.remote_control.takeoff(
            TakeoffRequest(
                sn="YOUR_DEVICE_SN",
                latitude=47.3769,
                longitude=8.5417,
                altitude=100.0,
            )
        )

        # Return-to-home
        await client.remote_control.return_to_home(
            ReturnToHomeRequest(sn="YOUR_DEVICE_SN")
        )


asyncio.run(main())
```

### Option B: long-lived singleton in a web app

```python
# app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from client_sdk import ZequentClient


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.zequent = ZequentClient.from_env()
    await app.state.zequent.__aenter__()
    yield
    await app.state.zequent.__aexit__(None, None, None)


app = FastAPI(lifespan=lifespan)


@app.post("/drone/{sn}/takeoff")
async def takeoff(sn: str, lat: float, lon: float, alt: float):
    from client_sdk import TakeoffRequest
    return await app.state.zequent.remote_control.takeoff(
        TakeoffRequest(sn=sn, latitude=lat, longitude=lon, altitude=alt)
    )
```

### Option C: explicit configuration (no env vars)

```python
from client_sdk import ZequentClient
from client_sdk.config.service_config import ServiceConfig

async with ZequentClient(
    connector_config=ServiceConfig(service_name="connector", host="c.prod.example.com", port=8010),
    remote_control_config=ServiceConfig(service_name="remote-control", host="rc.prod.example.com", port=8002),
    mission_autonomy_config=ServiceConfig(service_name="mission-autonomy", host="ma.prod.example.com", port=8004),
    live_data_config=ServiceConfig(service_name="live-data", host="ld.prod.example.com", port=8003),
) as client:
    ...
```

---

## Complete example: a FastAPI drone gateway

```python
import asyncio
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends, Request
from client_sdk import (
    ZequentClient,
    TakeoffRequest, GoToRequest, ReturnToHomeRequest,
    DockOperationRequest, StreamTelemetryRequest,
)


@asynccontextmanager
async def lifespan(app: FastAPI):
    client = ZequentClient.from_env()
    await client.__aenter__()
    app.state.zequent = client
    try:
        yield
    finally:
        await client.__aexit__(None, None, None)


app = FastAPI(lifespan=lifespan)


def get_client(request: Request) -> ZequentClient:
    return request.app.state.zequent


@app.post("/api/drone/{sn}/takeoff")
async def takeoff(
    sn: str, lat: float, lon: float, alt: float,
    client: ZequentClient = Depends(get_client),
):
    return await client.remote_control.takeoff(
        TakeoffRequest(sn=sn, latitude=lat, longitude=lon, altitude=alt)
    )


@app.post("/api/drone/{sn}/goto")
async def goto(
    sn: str, lat: float, lon: float, alt: float,
    client: ZequentClient = Depends(get_client),
):
    return await client.remote_control.go_to(
        GoToRequest(sn=sn, latitude=lat, longitude=lon, altitude=alt)
    )


@app.post("/api/drone/{sn}/return-home")
async def return_home(sn: str, client: ZequentClient = Depends(get_client)):
    return await client.remote_control.return_to_home(ReturnToHomeRequest(sn=sn))


@app.post("/api/drone/{sn}/dock/open-cover")
async def open_cover(sn: str, client: ZequentClient = Depends(get_client)):
    return await client.remote_control.open_cover(DockOperationRequest(sn=sn))


@app.get("/api/drone/{sn}/telemetry")
async def telemetry_stream(sn: str, client: ZequentClient = Depends(get_client)):
    # Collect the first 5 telemetry frames and return them
    frames = []
    enough = asyncio.Event()

    def on_frame(frame):
        frames.append(str(frame))
        if len(frames) >= 5:
            enough.set()

    async with client.live_data.stream_telemetry(StreamTelemetryRequest(sn=sn), on_frame):
        await asyncio.wait_for(enough.wait(), timeout=30)
    return frames
```

---

## Step 4: Run platform services

You need the Zequent platform services running from the published container images. See [Zequent Documentation](../README.md) for the compose file.

```bash
docker compose -f docker-compose.customer.yml up -d
```

---

## Next steps

- [Applications & Skills](../concepts/applications-and-skills.md) — running Skills and Applications from your code
- [Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy-python.md) — executions, Applications & schedulers
- [Waypoint Missions](WAYPOINT_MISSIONS.md) — flying a waypoint route
- [Remote Control](REMOTE_CONTROL_PYTHON.md) — flight ops, dock ops, manual control
- [Connector reference](CONNECTOR_PYTHON.md) — assets, organizations, schedulers, technical config
- [Configuration reference](CONFIGURATION_PYTHON.md) — every env var the SDK reads
- [Customer example](CUSTOMER_EXAMPLE_PYTHON.md) — full working FastAPI sample
- [Asyncio patterns](ASYNCIO.md) — lifecycles, streaming, cancellation
