# Zequent Client SDK (Python) - Customer Example

A complete, runnable example of a small FastAPI service that uses the Python Client SDK to expose a REST API for drone operations. This is the Python counterpart of [CUSTOMER_EXAMPLE.md](CUSTOMER_EXAMPLE.md).

---

## Project layout

```
my-drone-gateway/
├── pyproject.toml
├── .env
└── app/
    ├── __init__.py
    ├── main.py
    └── models.py
```

## `pyproject.toml`

```toml
[project]
name = "my-drone-gateway"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "zqnt-client-sdk>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "python-dotenv>=1.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

Install with:

```bash
uv sync
```

## `.env`

```bash
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002
MISSION_AUTONOMY_SERVICE_HOST=localhost
MISSION_AUTONOMY_SERVICE_PORT=8004
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003
```

## `app/models.py`

Lightweight request models — keep your wire format separate from the SDK's:

```python
from pydantic import BaseModel, Field


class TakeoffBody(BaseModel):
    latitude: float = Field(..., ge=-90, le=90)
    longitude: float = Field(..., ge=-180, le=180)
    altitude: float = Field(..., ge=0, le=500)


class GoToBody(TakeoffBody):
    pass


class ExecuteSkillBody(BaseModel):
    application_id: str
    skill_id: str
    application_version: str | None = None
    parameters: dict = Field(default_factory=dict)
```

## `app/main.py`

```python
import logging
from contextlib import asynccontextmanager

from dotenv import load_dotenv
from fastapi import Depends, FastAPI, HTTPException, Request

from client_sdk import (
    GoToRequest,
    ReturnToHomeRequest,
    TakeoffRequest,
    ZequentClient,
    ZequentClientError,
    ZequentRetryExhaustedError,
)

from .models import ExecuteSkillBody, GoToBody, TakeoffBody

load_dotenv()
logging.basicConfig(level=logging.INFO)
log = logging.getLogger("drone-gateway")


@asynccontextmanager
async def lifespan(app: FastAPI):
    client = ZequentClient.from_env()
    await client.__aenter__()
    app.state.zequent = client
    log.info("Zequent client connected")
    try:
        yield
    finally:
        await client.__aexit__(None, None, None)
        log.info("Zequent client closed")


app = FastAPI(title="Drone Gateway", lifespan=lifespan)


def get_client(request: Request) -> ZequentClient:
    return request.app.state.zequent


# ----------------------------------------------------------------------
# Remote Control
# ----------------------------------------------------------------------

@app.post("/drones/{sn}/takeoff")
async def takeoff(
    sn: str, body: TakeoffBody,
    client: ZequentClient = Depends(get_client),
):
    try:
        return await client.remote_control.takeoff(
            TakeoffRequest(sn=sn, **body.model_dump())
        )
    except ZequentRetryExhaustedError as e:
        raise HTTPException(503, f"Service unavailable: {e}") from e
    except ZequentClientError as e:
        raise HTTPException(502, str(e)) from e


@app.post("/drones/{sn}/goto")
async def goto(
    sn: str, body: GoToBody,
    client: ZequentClient = Depends(get_client),
):
    return await client.remote_control.go_to(GoToRequest(sn=sn, **body.model_dump()))


@app.post("/drones/{sn}/return-home")
async def return_home(sn: str, client: ZequentClient = Depends(get_client)):
    return await client.remote_control.return_to_home(ReturnToHomeRequest(sn=sn))


# ----------------------------------------------------------------------
# Mission Autonomy — missions & tasks
# See: WAYPOINT_MISSIONS.md
# ----------------------------------------------------------------------

@app.post("/drones/{sn}/skills/execute")
async def execute_skill(
    sn: str, body: ExecuteSkillBody, client: ZequentClient = Depends(get_client),
):
    """Run a named Skill from a deployed Application against one asset."""
    return await client.mission_autonomy.execute_application(
        asset_sn=sn,
        application_id=body.application_id,
        skill_id=body.skill_id,
        application_version=body.application_version,
        parameters=body.parameters,
    )


@app.get("/executions/{execution_id}")
async def get_execution(execution_id: str, client: ZequentClient = Depends(get_client)):
    return await client.mission_autonomy.get_skill_execution(execution_id)


@app.post("/executions/{execution_id}/cancel")
async def cancel_execution(execution_id: str, client: ZequentClient = Depends(get_client)):
    return await client.mission_autonomy.cancel_skill_execution(execution_id)


# ----------------------------------------------------------------------
# Live Data (server-streaming)
# ----------------------------------------------------------------------

@app.get("/drones/{sn}/telemetry")
async def telemetry_window(
    sn: str, frames: int = 5, client: ZequentClient = Depends(get_client),
):
    """Collect the next N telemetry frames and return them as a list."""
    out = []
    async for frame in client.live_data.stream_telemetry(asset_sn=sn):
        out.append(frame)
        if len(out) >= frames:
            break
    return out
```

## Run it

```bash
uv run uvicorn app.main:app --reload
```

Test:

```bash
curl -X POST http://localhost:8000/drones/DOCK-1/takeoff \
  -H 'content-type: application/json' \
  -d '{"latitude":47.3769,"longitude":8.5417,"altitude":100}'
```

---

## Notes for production

- Use a long-lived `ZequentClient` per process (the lifespan does this) — never instantiate one per request.
- Wrap each handler in proper error mapping; the example covers the SDK's two main exception types.
- For high-fanout streaming endpoints, prefer FastAPI's `StreamingResponse` to push frames as they arrive instead of buffering.
- Add structured request logging that captures `sn` + `task_id` so you can correlate with platform logs.
- Don't forget `--workers 1` if you rely on a single `ZequentClient` in `app.state` — for multi-worker setups, each worker creates its own client instance, which is fine.
- Configure TLS via custom channels (see [CONFIGURATION_PYTHON.md](CONFIGURATION_PYTHON.md)) when deploying outside a private network.
