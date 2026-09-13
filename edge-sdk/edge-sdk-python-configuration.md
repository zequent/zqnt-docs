# Edge SDK (Python) — Configuration

The Python Edge SDK is configured exclusively via **environment variables**, centralized in `EdgeAdapterConfig`. Call `EdgeAdapterConfig.from_env()` to build one, then `.runtime()` to get a connected `EdgeAdapterRuntime` — see the [Quickstart](edge-sdk-python-quickstart.md).

For Java/Quarkus configuration see [edge-sdk-configuration.md](edge-sdk-configuration.md).

---

## `EdgeAdapterConfig` environment variables

| Variable | Description | Default |
|----------|--------------|---------|
| `GRPC_HOST` | Bind address for this adapter's own gRPC server | `0.0.0.0` |
| `GRPC_PORT` | Bind port for this adapter's own gRPC server | `50051` |
| `CONNECTOR_HOST` | Connector service hostname | `localhost` |
| `CONNECTOR_PORT` | Connector service gRPC port | `50053` |
| `TELEMETRY_HOST` | Live Data service hostname | `localhost` |
| `TELEMETRY_PORT` | Live Data service gRPC port | `50052` |
| `MISSION_AUTONOMY_HOST` | Mission Autonomy service hostname | `localhost` |
| `MISSION_AUTONOMY_PORT` | Mission Autonomy service gRPC port | `50054` |
| `ADAPTER_SN` | Serial number this adapter logs under; each telemetry frame also carries its own asset SN, so this doesn't need to match for multi-asset adapters | `""` |
| `LOG_LEVEL` | Python log level name | `INFO` |
| `LOG_FORMAT` | `json` or `text` | `json` |

**The library's built-in port defaults do not match the platform's real service ports.** Always set `CONNECTOR_PORT=8010`, `TELEMETRY_PORT=8003`, and `MISSION_AUTONOMY_PORT=8004` (or your deployment's actual ports) explicitly — don't rely on the defaults above.

```python
from edge_sdk import EdgeAdapterConfig

config = EdgeAdapterConfig.from_env()
async with config.runtime() as runtime:
    ...
```

For tests or explicit configuration, construct `EdgeAdapterConfig(...)` directly instead of calling `from_env()` — every field has a keyword-argument equivalent.

---

## Optional: automatic Redis service-discovery registration

If set, the SDK registers this adapter's endpoint in Redis on startup (`online=True`) and marks it offline on shutdown, so the platform's client-side load balancer can discover it dynamically.

| Variable | Description | Default |
|----------|--------------|---------|
| `EDGE_ENDPOINT` | gRPC endpoint advertised to the platform, e.g. `grpc://my-adapter.internal:9001` | unset (registration off) |
| `ASSET_TYPE` | Proto-style asset type name, e.g. `ASSET_TYPE_AIRCRAFT` | unset |
| `ASSET_VENDOR` | Proto-style vendor name, e.g. `ASSET_VENDOR_MAVLINK` | unset |
| `REDIS_URL` | Redis connection URL | `redis://localhost:6379` |

Registration only activates when `EDGE_ENDPOINT`, `ASSET_TYPE`, and `ASSET_VENDOR` are all set — `EdgeAdapterConfig.runtime()`'s `serve()` builds the `RegistrationConfig` for you automatically in that case. To configure it directly:

```python
from edge_sdk import RegistrationConfig

registration = RegistrationConfig.from_env()  # reads EDGE_ENDPOINT / ASSET_TYPE / ASSET_VENDOR / REDIS_URL
```

**Redis key format: `zqnt:edge-endpoints:{VENDOR}`.** This does *not* currently match the Java SDK's
own `CacheKeys.EDGE_ENDPOINTS` key (`edge-endpoints:{vendor}`, no `zqnt:` prefix) or the Go SDK's
`discovery` package (which explicitly mirrors Java's un-prefixed key, per its own code comment). This
is a real, confirmed cross-language mismatch in the current Python Edge SDK, not a documentation
choice — a Python adapter's `EDGE_ENDPOINT` registration writes to a different Redis key than what
Java-side code reads from, so it will not be discovered the way a Java or Go adapter's registration
would be.

---

## Logging

`EdgeAdapterRuntime` calls `logging.basicConfig(...)` for you based on `LOG_LEVEL`/`LOG_FORMAT` — you don't need to configure it yourself unless you want something different. `LOG_FORMAT=json` uses `python-json-logger` if installed, falling back to plain text otherwise.

```python
import logging
logging.getLogger("edge_sdk").setLevel(logging.DEBUG)
```

---

## TLS / authentication

**Not supported, at any layer.** Confirmed directly in source: `ConnectorClient.connect()`,
`TelemetryPublisher`'s internal stream setup, and `MissionAutonomyClient.connect()` each construct
their gRPC channel with a hardcoded `grpc.aio.insecure_channel(host:port)` call — none of these
classes accepts a `channel`, credentials, or any other override in their constructor or `connect()`.
There is currently no way to reach the platform over TLS, or attach per-call auth metadata, from the
Python Edge SDK's own client classes. If your deployment requires TLS between the adapter and the
platform, terminate it at a sidecar/proxy in front of the platform services instead.

---

## Putting it together

```python
import asyncio
from edge_sdk import EdgeAdapterConfig

from .adapter import MyDeviceAdapter

async def main():
    config = EdgeAdapterConfig.from_env()
    async with config.runtime() as runtime:
        adapter = MyDeviceAdapter()
        await runtime.serve(adapter)

asyncio.run(main())
```
