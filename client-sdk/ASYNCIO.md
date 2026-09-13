# Zequent Client SDK (Python) - Asyncio Patterns

The Java Client SDK integrates tightly with Quarkus / CDI / Mutiny. The Python Client SDK is plain `asyncio` + `grpc.aio`. This document covers the lifecycle, streaming, cancellation, and error patterns that make the Python SDK pleasant to use in real applications.

This is the Python counterpart of [SPRING_BOOT.md](SPRING_BOOT.md).

---

## Lifecycle: one client per process

`ZequentClient` is **stateful** — it owns three `grpc.aio` channels. Create exactly one per process and reuse it:

```python
async with ZequentClient.from_env() as client:
    await do_work(client)
```

Under the hood:

1. `__aenter__()` creates the three channels and lazily-initialised stubs.
2. `__aexit__()` closes them in reverse order.

For a script, the `async with` form is enough. For a long-running server (FastAPI, aiohttp, custom asyncio loop), wire it into a **lifespan** so it stays alive between requests:

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

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
```

For a Click / Typer CLI:

```python
@click.command()
def takeoff(sn: str):
    asyncio.run(_takeoff(sn))

async def _takeoff(sn: str):
    async with ZequentClient.from_env() as client:
        await client.remote_control.takeoff(TakeoffRequest(sn=sn))
```

---

## Concurrency

`grpc.aio` channels are HTTP/2-multiplexed and re-entrant. You can issue many concurrent calls on a single `ZequentClient`:

```python
async with ZequentClient.from_env() as client:
    drones = ["DOCK-1", "DOCK-2", "DOCK-3"]
    results = await asyncio.gather(*[
        client.remote_control.takeoff(TakeoffRequest(sn=sn)) for sn in drones
    ])
```

There's no need for connection pools or thread pools. **Do not** wrap calls in `asyncio.to_thread` — they're already non-blocking.

---

## Cancellation

Cancelling the calling task cancels the underlying gRPC call. Use it for timeouts or for early termination of streaming reads:

```python
try:
    async with asyncio.timeout(2.0):
        await client.mission_autonomy.get_task("slow-task-id")
except TimeoutError:
    log.warning("get_task timed out")
```

For streams, simply `break` out of the iterator or let the surrounding context manager exit:

```python
async for frame in client.live_data.stream_telemetry(asset_sn="DOCK-1"):
    if frame.battery_percentage < 20:
        break    # cancels the underlying gRPC stream
```

---

## Streaming

Server-streaming RPCs return an async iterator. Cancellation, back-pressure, and clean-up are managed by the iterator protocol — you don't need to call `.close()` on a stream you've already exhausted or broken out of.

For long-lived streams that you want to control explicitly (start/stop from different coroutines), use `StreamHandle`:

```python
from client_sdk import StreamHandle

handle: StreamHandle = await client.live_data.stream_telemetry(sn="DOCK-1", on_data=consume)
# ... later, from anywhere:
await handle.close()
```

For bi-directional streams (e.g. manual control), use the `manual_control_session` async context:

```python
async with client.remote_control.manual_control_session(sn="DOCK-1") as session:
    await session.send(ManualControlInput(throttle=0.5, yaw=0.1))
    response = await session.recv()
```

---

## Error handling

All client errors derive from `ZequentClientError`:

| Exception                       | When it's raised                                                |
|---------------------------------|-----------------------------------------------------------------|
| `ZequentClientError`            | Generic base — catch this if you don't care about specifics    |
| `ZequentRetryExhaustedError`    | Retries gave up; the underlying gRPC call kept failing          |
| `CircuitBreakerOpen`            | The breaker for this method is open; refusing to send the call  |
| `grpc.aio.AioRpcError`          | Low-level gRPC errors that don't get wrapped (rare)             |

Recommended pattern:

```python
from client_sdk import ZequentClientError, ZequentRetryExhaustedError

try:
    await client.remote_control.takeoff(req)
except ZequentRetryExhaustedError as e:
    # Likely: backend down. Surface as 503.
    raise HTTPException(503, str(e)) from e
except ZequentClientError as e:
    # Anything else from the SDK
    raise HTTPException(502, str(e)) from e
```

Don't catch `Exception` broadly — let programmer errors (validation, type errors) surface.

---

## Resilience tuning

Override the resilience policy when you have requirements that differ from the defaults:

```python
from client_sdk.config.resilience import ResilienceConfig

resilience = ResilienceConfig(
    max_attempts=3,
    initial_backoff_ms=100,
    max_backoff_ms=2_000,
    backoff_multiplier=2.0,
    breaker_failure_threshold=5,
    breaker_reset_seconds=15,
)

async with ZequentClient(config) as client:
    ...
```

Retries apply only to **unary** RPCs and only on retryable gRPC status codes. See [CONFIGURATION_PYTHON.md](CONFIGURATION_PYTHON.md) for the full list.

---

## Testing

The Python SDK is built around `MagicMock`-friendly stubs. The recommended pattern in tests:

```python
from unittest.mock import AsyncMock, MagicMock
from client_sdk.remote_control.client import RemoteControlClient


async def test_takeoff_sends_correct_request():
    stub = MagicMock()
    stub.Takeoff = AsyncMock(return_value=...)

    client = RemoteControlClient(stub=stub)
    await client.takeoff(TakeoffRequest(sn="DOCK-1"))

    stub.Takeoff.assert_awaited_once()
    sent = stub.Takeoff.await_args.args[0]
    assert sent.sn == "DOCK-1"
```

For integration tests, spin up the platform services (compose) and use a real `ZequentClient.from_env()`.

---

## Comparison to the Java SDK

| Concern                    | Java                                            | Python                                          |
|----------------------------|-------------------------------------------------|-------------------------------------------------|
| Concurrency                | `CompletableFuture<T>`, Mutiny                  | `async def` / `await` + `grpc.aio`              |
| DI                         | CDI (`@Inject ZequentClient`)                   | Pass the client manually (FastAPI / DI of choice)|
| Configuration              | `application.properties` + env                  | Env (`from_env()`) or explicit `ServiceConfig` |
| Lifecycle                  | CDI `@PostConstruct` / `@PreDestroy`            | `async with` / lifespan hook                    |
| Streaming                  | `Multi<T>` (Mutiny)                             | Async iterator (`async for`)                    |
| Retries                    | SmallRye fault tolerance                        | Built-in `GrpcResilience`                       |
| Error type                 | `ErrorInfo` on the response (no exceptions)     | `ZequentClientError`                            |

The Python SDK is intentionally lean — there is no DI, no annotations, no codegen step beyond `generate_protos.sh`. If something feels missing, check whether it can be expressed with stdlib `asyncio` + the patterns in this document.

---

## See also

- [Quickstart](QUICKSTART_PYTHON.md) — first 10 minutes
- [Configuration](CONFIGURATION_PYTHON.md) — every env var the SDK reads
- [Customer example](CUSTOMER_EXAMPLE_PYTHON.md) — full FastAPI sample
- SDK README - package-level reference
