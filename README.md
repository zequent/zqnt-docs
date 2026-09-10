# Zequent Documentation

Zequent is a platform for connecting, monitoring, and controlling remote assets — drones, docks, ground vehicles, and other edge devices — from your own applications, and for building autonomous, multi-step operations for them without writing device-specific code.

This public documentation is for external developers and integration teams. It focuses on:

- using the Client SDKs from customer applications
- creating missions and waypoint tasks and running them on connected assets
- building custom edge adapters with the Edge SDKs
- running Zequent platform services from published container images
- deploying supported edge adapter images

## Start Here

| Goal | Documentation |
| --- | --- |
| Understand Assets vs SubAssets, and how telemetry identifies its source | [Assets & Sub-Assets](concepts/assets-and-sub-assets.md) |
| Use Zequent from a Java application | [Java Client SDK Quickstart](client-sdk/QUICKSTART.md) |
| Fly a waypoint mission from your own application | [Waypoint Missions](client-sdk/WAYPOINT_MISSIONS.md) |
| Use Zequent from a Python application | [Python Client SDK Quickstart](client-sdk/QUICKSTART_PYTHON.md) |
| Use Zequent from a Go application | [Go Client SDK Quickstart](client-sdk/QUICKSTART_GO.md) |
| Configure a customer application / deployment | [Client SDK Configuration](client-sdk/CONFIGURATION.md) |
| Look up assets, organizations, schedulers from your app | [Connector (Java)](client-sdk/CONNECTOR.md) / [Connector (Python)](client-sdk/CONNECTOR_PYTHON.md) / [Connector (Go)](client-sdk/CONNECTOR_GO.md) |
| Build a custom Java edge adapter | [Java Edge SDK Quickstart](edge-sdk/edge-sdk-quickstart.md) |
| Build a custom Python edge adapter | [Python Edge SDK Quickstart](edge-sdk/edge-sdk-python-quickstart.md) |
| Build a custom Go edge adapter | [Go Edge SDK Quickstart](edge-sdk/edge-sdk-go-quickstart.md) — older API surface, see the doc's status note |
| Configure an edge adapter | [Edge SDK Configuration](edge-sdk/edge-sdk-configuration.md) |
| Deploy a ready-made adapter | [DJI](edge-sdk/edge-sdk-dji-adapter-deployment.md) · [MAVLink](edge-sdk/edge-sdk-mavlink-adapter-deployment.md) · [Sapient](edge-sdk/edge-sdk-sapient-adapter-deployment.md) · [RNS](edge-sdk/edge-sdk-rns-adapter-deployment.md) · [Betaflight](edge-sdk/edge-sdk-betaflight-adapter-deployment.md) · [AI Adapter](edge-sdk/edge-sdk-ai-adapter-deployment.md) |


## What You Can Build

- **Direct control** — takeoff, go-to, return-to-home, dock open/close, camera and gimbal control, and live joystick-style manual control, called directly from your application via the Client SDK.
- **Missions & Tasks** — define a mission, attach waypoint tasks to it, and start, pause, resume or stop them from your own code. See [Waypoint Missions](client-sdk/WAYPOINT_MISSIONS.md).
- **Live telemetry & detections** — subscribe to real-time position, battery, and sensor telemetry, and AI detection results, streamed from every connected asset.
- **Live video** — start/stop live video streams from a connected asset's camera and view them in the Admin Console or your own player.
- **Custom hardware integrations** — build a new edge adapter with the Edge SDK for any device that isn't already supported, using the same command/telemetry contract every built-in adapter uses.

## Customer Applications

Customer applications normally use the Client SDK and connect to the platform service endpoints exposed by your deployment.

| SDK | Main docs |
| --- | --- |
| Java Client SDK | [Quickstart](client-sdk/QUICKSTART.md), [Configuration](client-sdk/CONFIGURATION.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE.md) |
| Python Client SDK | [Quickstart](client-sdk/QUICKSTART_PYTHON.md), [Configuration](client-sdk/CONFIGURATION_PYTHON.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE_PYTHON.md) |
| Go Client SDK | [Quickstart](client-sdk/QUICKSTART_GO.md), [Connector](client-sdk/CONNECTOR_GO.md) — no built-in retry/circuit-breaker/Stork layer, see the quickstart's "What this SDK deliberately does not do" |

## Platform Service Images

Zequent platform services are run from published container images. Use versioned tags for production deployments.

| Component | Image | Default port | Customer-facing purpose |
| --- | --- | ---: | --- |
| Connector Service | `ghcr.io/zequent/connector-service:1.3.1` | `8010` | System of record: assets, organizations, missions and tasks, schedulers, technical config, telemetry persistence |
| Remote Control Service | `ghcr.io/zequent/remote-control-service:1.3.1` | `8002` | Direct asset commands such as takeoff, go-to, return-to-home, dock, camera, and manual-control commands |
| Live Data Service | `ghcr.io/zequent/live-data-service:1.3.1` | `8003` | Live telemetry, detections, and task-progress notification streams |
| Mission Autonomy Service | `ghcr.io/zequent/mission-autonomy-service:1.3.1` | `8004` | Executes mission tasks and manages schedulers |
| Admin Console API | `ghcr.io/zequent/admin-console-service:1.3.1` | `8005` | HTTP/WebSocket API for the Admin Console |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v1.3.3` | `3001` | Browser UI: asset monitoring, live streams, manual control, and mission planning |

The Admin Console UI's image is `zqnt-platform-console`, not `zqnt-admin-console-dashboard` — that name is not a real published package, and pulling it fails. Its version line (`vX.Y.Z`) is independent of the core services' `1.3.x` line.

Platform services also require **Postgres (TimescaleDB)** and **Redis** — see [docker-compose.customer.yml](docker-compose.customer.yml). A runnable copy of the same file also lives at `core/docker-compose.customer.yml`, alongside `docker-compose.local.yml`/`docker-compose.env.yml`, for working directly in the monorepo — keep both copies in sync if you edit one.

## Edge Adapter Images

Use these adapter images when you want a ready-made integration. Use the Edge SDK when you need to build a custom adapter.

| Adapter | Image | Status | Notes |
| --- | --- | --- | --- |
| DJI | `ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0` | Available | DJI dock/drone integration. [Deployment guide](edge-sdk/edge-sdk-dji-adapter-deployment.md) |
| MAVLink | `ghcr.io/zequent/zqnt-adapter-mavlink:1.3.0` | Available | PX4/ArduPilot vehicles via MAVSDK. [Deployment guide](edge-sdk/edge-sdk-mavlink-adapter-deployment.md) |
| Sapient | `ghcr.io/zequent/zqnt-adapter-sapient:1.3.0` | Available | Bridges TCP SAPIENT edge nodes to gRPC. [Deployment guide](edge-sdk/edge-sdk-sapient-adapter-deployment.md) |
| RNS | No versioned release yet (`latest` only) | Source only | Early-stage — implements asset registration and vendor custom commands only. [Deployment guide](edge-sdk/edge-sdk-rns-adapter-deployment.md) |
| Betaflight | No published image yet | Source only | Serial/USB flight-controller integration; no container packaging yet. [Deployment guide](edge-sdk/edge-sdk-betaflight-adapter-deployment.md) |
| AI Adapter | No published image yet | Early access | RTMP/RTSP video → YOLO detection → georeferenced results, with optional gimbal re-aim. Uses the standard Edge SDK adapter pattern. [Deployment guide](edge-sdk/edge-sdk-ai-adapter-deployment.md) |

The DJI image is `zqnt-edge-adapter-dji`, not `dji-adapter` — that older package name still exists but is stale/abandoned (only a floating `latest`, no versioned releases). "Source only" / "Early access" adapters are real, working code you can run today with `uv run` — they just don't have a versioned container image yet. Build your own image from the adapter's own `Dockerfile` where one exists, or contact Zequent about early access to a build.

### Load-test / fleet simulator

`ghcr.io/zequent/zqnt-simulator:1.3.3` — a Go tool that simulates a fleet of edge adapters against
the live stack, for load-testing the Live Data gRPC telemetry ingest path and exercising
remote-control/admin-console fleet flows without real hardware. Optional, enabled via the
`simulator` Compose profile — not part of a normal customer deployment. See the repo's own
`simulator/README.md` for the full env var reference.

## Deployment Configuration

Container deployments use one deployment-local `.env` file referenced by [docker-compose.customer.yml](docker-compose.customer.yml). Start from [.env.customer](.env.customer) — copy it to `.env` next to the compose file and fill in every `<PLACEHOLDER>` (database/Redis passwords, your Ed25519 signing key, your license key, public dashboard URLs) before starting the stack. It contains no Zequent-internal credentials — every value is either a safe structural default or a placeholder only you can fill in.

```yaml
services:
  connector-service:
    image: ghcr.io/zequent/connector-service:1.3.1
    env_file:
      - .env
```

Use the same `env_file: .env` pattern for Zequent service images, Admin Console images, adapter images, and customer application containers.

Do not commit `.env` files. Keep credentials, database/broker settings, stream URLs, license keys, and deployment-specific hostnames in your deployment environment or secret manager.

Start the stack with:

```bash
docker compose -f docker-compose.customer.yml up -d
```

Optional adapter images are enabled through Compose profiles, for example:

```bash
docker compose -f docker-compose.customer.yml --profile edge-dji up -d
```

See [Client SDK Configuration](client-sdk/CONFIGURATION.md) for the full `.env` reference, including the required Postgres/Redis and licensing variables.

## Licensing

Every platform service enforces an activated license before it will perform protected operations — a fresh deployment with no license activated will reject most requests. Licenses are organization- and seat-based: one license covers one organization, and each platform user you create consumes one of that organization's seats.

1. Zequent issues you a license key when you purchase a subscription.
2. Activate it once, from the Admin Console, against `https://api.zequent.com` (the default `LICENSE_SERVER_URL` in production deployments).
3. Services automatically refresh their license lease afterward — no further manual steps.

See [Client SDK Configuration](client-sdk/CONFIGURATION.md) for the `LICENSING_*` environment variables.

## Admin Console

The Admin Console is split into an API image and a UI image.

| Component | Default local URL |
| --- | --- |
| Admin Console UI | `http://localhost:3001` |
| Admin Console API | `http://localhost:8005` |

The Admin Console provides browser workflows for asset monitoring, telemetry, mission planning, remote control, live streams, adapter management, licensing, and service health.


## SDK Requirements

| SDK | Requirements |
| --- | --- |
| Java Client SDK / Java Edge SDK | Java 25, Maven 3.9+ recommended, Quarkus 3.x for Quarkus applications |
| Python Client SDK / Python Edge SDK | Python 3.12+, `uv` recommended, `grpc.aio` |
| Go Client SDK / Go Edge SDK | Go 1.26+ (client) / Go 1.24+ (edge). Both modules are private (`github.com/Zequent/zqnt-client-sdk-go`, `github.com/Zequent/zqnt-edge-sdk-go`) — see each quickstart's `go env -w GONOSUMDB`/`GONOPROXY` step. The Go Edge SDK is on an older API surface than the Java/Python Edge SDKs — see its [overview](edge-sdk/edge-sdk-go-overview.md). |

## Package Access

If you consume private Zequent packages, configure access to the relevant package registry before building your customer application or adapter.

For Maven packages, configure your `~/.m2/settings.xml` with a token that has package read access. For Python packages, configure `uv`/`pip` with a token that has read access to the package index or private Git repository you were given access to.

## Production Notes

- Run platform services and provided adapters from published container images.
- Use versioned image tags for production, not `:latest`.
- Keep secrets outside Git.
- Use TLS and deployment-managed secrets for production environments.
- Use the Client SDKs for customer applications and the Edge SDKs for custom adapters.
