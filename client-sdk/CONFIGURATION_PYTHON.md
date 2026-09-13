# Zequent Client SDK (Python) - Configuration

The Python Client SDK is configured exclusively via **environment variables** read by `ZequentClient.from_env()`, or by passing a `ServiceConfig` per service explicitly. There is no `application.properties` equivalent and no DI container.

For Java/Quarkus configuration see [CONFIGURATION.md](CONFIGURATION.md).

---

## Service endpoints

| Variable                          | Default     | Description                                  |
|-----------------------------------|-------------|----------------------------------------------|
| `CONNECTOR_SERVICE_HOST`          | `localhost` | Hostname of the Connector service            |
| `CONNECTOR_SERVICE_PORT`          | `8010`      | gRPC port                                    |
| `REMOTE_CONTROL_SERVICE_HOST`     | `localhost` | Hostname of the Remote Control service       |
| `REMOTE_CONTROL_SERVICE_PORT`     | `8002`      | gRPC port                                    |
| `MISSION_AUTONOMY_SERVICE_HOST`   | `localhost` | Hostname of the Mission Autonomy service     |
| `MISSION_AUTONOMY_SERVICE_PORT`   | `8004`      | gRPC port                                    |
| `LIVE_DATA_SERVICE_HOST`          | `localhost` | Hostname of the Live Data service            |
| `LIVE_DATA_SERVICE_PORT`          | `8003`      | gRPC port                                    |

Unlike the Python Edge SDK's own client classes (whose built-in port defaults don't match the
platform's real ports — see [Edge SDK Configuration](../edge-sdk/edge-sdk-python-configuration.md)),
these Client SDK defaults are the platform's actual ports directly — no override needed for a local
Compose deployment.

```python
from client_sdk import ZequentClient

async with ZequentClient.from_env() as client:
    ...
```

Each of the four services above also accepts four more env vars with the same prefix, for TLS and
Kubernetes service discovery — see [TLS, auth, and custom channels](#tls-auth-and-custom-channels)
below, including its [Stork service discovery](#stork-service-discovery-kubernetes) subsection:

| Suffix | Default | Description |
| --- | --- | --- |
| `_USE_PLAINTEXT` | `true` | Set to `false` for TLS |
| `_USE_STORK` | `false` | Enable Stork-based service discovery instead of a fixed host/port |
| `_STORK_NAME` | `<service-name>-service` | Stork service name to resolve, when `_USE_STORK=true` |
| `_LOAD_BALANCER` | `ROUND_ROBIN` | Load-balancer strategy when `_USE_STORK=true` |

---

## Programmatic configuration

When env vars aren't a fit (multi-tenant apps, dynamic endpoints, tests):

```python
from client_sdk import ZequentClient
from client_sdk.config.service_config import ServiceConfig

async with ZequentClient(
    connector_config=ServiceConfig(service_name="connector", host="c.example.com", port=8010),
    remote_control_config=ServiceConfig(service_name="remote-control", host="rc.example.com", port=8002),
    mission_autonomy_config=ServiceConfig(service_name="mission-autonomy", host="ma.example.com", port=8004),
    live_data_config=ServiceConfig(service_name="live-data", host="ld.example.com", port=8003),
) as client:
    ...
```

`ZequentClient.from_env()` builds every `ServiceConfig` from the environment. To take the env
defaults and override one service, construct that one `ServiceConfig` yourself and pass all four
explicitly as above.

---

## Resilience configuration

Unary RPCs are wrapped with retry + circuit-breaker policies. `resilience` is a read-only property
set once at construction — either automatically from environment variables (see below) or by
passing a `ResilienceConfig` yourself:

```python
from client_sdk import ZequentClient
from client_sdk.config.resilience import ResilienceConfig

async with ZequentClient(
    resilience=ResilienceConfig(
        max_retry_attempts=5,
        retry_delay_millis=200,
        circuit_breaker_failure_threshold=10,
        circuit_breaker_wait_duration_millis=30_000,
        connection_timeout_seconds=30,
        request_timeout_seconds=60,
    ),
    connector_config=...,
    remote_control_config=...,
    mission_autonomy_config=...,
    live_data_config=...,
) as client:
    ...
```

`ZequentClient.from_env()` builds the same `ResilienceConfig` from env vars — the exact names the
Java SDK uses:

| Variable | Default | Maps to |
| --- | --- | --- |
| `ZEQUENT_MAX_RETRY_ATTEMPTS` | `3` | `max_retry_attempts` |
| `ZEQUENT_RETRY_DELAY_MS` | `1000` | `retry_delay_millis` |
| `ZEQUENT_CIRCUIT_BREAKER_THRESHOLD` | `5` | `circuit_breaker_failure_threshold` |
| `ZEQUENT_CIRCUIT_BREAKER_WAIT_MS` | `30000` | `circuit_breaker_wait_duration_millis` |
| `ZEQUENT_CONNECTION_TIMEOUT_SEC` | `30` | `connection_timeout_seconds` |
| `ZEQUENT_REQUEST_TIMEOUT_SEC` | `60` | `request_timeout_seconds` |

Retryable gRPC status codes:

- `UNAVAILABLE`
- `DEADLINE_EXCEEDED`
- `RESOURCE_EXHAUSTED`
- `ABORTED`

Non-retryable codes (propagated immediately):

- `INVALID_ARGUMENT`
- `FAILED_PRECONDITION`
- `NOT_FOUND`
- `ALREADY_EXISTS`
- `PERMISSION_DENIED`
- `UNAUTHENTICATED`

Streaming RPCs do **not** retry transparently — the application must re-subscribe.

---

## Per-call deadlines

`grpc.aio` deadlines are honoured for every unary call. Set a per-call deadline by passing `timeout=` to the SDK method:

```python
await client.remote_control.takeoff(req, timeout=5.0)
```

`timeout` is in seconds. If unset, the SDK uses the default channel deadline (none).

---

## Logging

Standard `logging` namespaces:

- `client_sdk.zequent_client` — top-level lifecycle
- `client_sdk.grpc_.resilience` — retry + breaker decisions
- `client_sdk.{remote_control,mission_autonomy,live_data}.client` — per-RPC logs

```python
import logging
logging.basicConfig(level=logging.INFO)
logging.getLogger("client_sdk.grpc_.resilience").setLevel(logging.DEBUG)
```

The SDK never logs sensitive request bodies; only operation name + outcome + status code.

---

## TLS, auth, and custom channels

By default the SDK uses **insecure** gRPC channels for parity with local development. To use TLS or auth, pass a pre-built `grpc.aio.Channel` per service:

```python
from client_sdk import ZequentClient
from client_sdk.config.service_config import ServiceConfig

# TLS is selected per service with use_plaintext=False; the SDK builds the channel itself.
async with ZequentClient(
    connector_config=ServiceConfig(service_name="connector", host="c.example.com", port=443, use_plaintext=False),
    remote_control_config=ServiceConfig(service_name="remote-control", host="rc.example.com", port=443, use_plaintext=False),
    mission_autonomy_config=ServiceConfig(service_name="mission-autonomy", host="ma.example.com", port=443, use_plaintext=False),
    live_data_config=ServiceConfig(service_name="live-data", host="ld.example.com", port=443, use_plaintext=False),
) as client:
    ...
```

For per-call metadata (e.g. JWT bearers), pass `metadata=[(...)]` to SDK methods or attach a gRPC interceptor to the channel.

The same switch is available per service as an env var, without touching code —
`CONNECTOR_SERVICE_USE_PLAINTEXT=false`, `REMOTE_CONTROL_SERVICE_USE_PLAINTEXT=false`, etc., read by
`ZequentClient.from_env()` (see [Service endpoints](#service-endpoints) above).

### Stork service discovery (Kubernetes)

Each service also accepts `<PREFIX>_USE_STORK=true` (with `<PREFIX>_STORK_NAME` and
`<PREFIX>_LOAD_BALANCER` to override the defaults) to resolve the endpoint via Stork instead of a
fixed host/port — the same mechanism the Java SDK uses in Kubernetes, e.g.:

```bash
REMOTE_CONTROL_SERVICE_USE_STORK=true
REMOTE_CONTROL_SERVICE_STORK_NAME=remote-control-service
REMOTE_CONTROL_SERVICE_LOAD_BALANCER=ROUND_ROBIN
```

---

## Connection pooling

Each sub-client owns one `grpc.aio` channel. `grpc.aio` channels multiplex unbounded concurrent calls over HTTP/2, so a single `ZequentClient` instance scales to thousands of concurrent calls without further pooling.

**Do not create one `ZequentClient` per request** — keep it as an application singleton (per the lifespan pattern in the [Quickstart](QUICKSTART_PYTHON.md)).

---

## Environment for local development

The simplest setup uses the bundled compose file:

```bash
docker compose up -d
```

Then leave all the `*_HOST` / `*_PORT` env vars at their defaults — `localhost` and the ports above match the compose file exactly.

For Kubernetes, point each `*_SERVICE_HOST` at the in-cluster Service DNS name (e.g. `remote-control-service.zequent.svc.cluster.local`).
