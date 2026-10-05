# Zequent Documentation

Zequent is a platform for connecting, monitoring, and controlling remote assets — drones, docks, ground vehicles, and other edge devices — from your own applications, and for building autonomous, multi-step operations for them without writing device-specific code.

This public documentation is for external developers and integration teams. It focuses on:

- using the Client SDKs from customer applications
- building Applications and Skills — multi-step automations — and running them on connected assets
- building custom edge adapters with the Edge SDKs
- running Zequent platform services from published container images
- deploying supported edge adapter images

> **Versions.** This documentation covers Zequent **2.0.x**, the current long-term-support (LTS) line.
> **1.3.x has reached end of life**: it keeps running, but receives no fixes or support. Its
> documentation is kept unchanged in the files ending in `-1.3.md` (for example the
> [1.3 Java Client SDK Quickstart](client-sdk/QUICKSTART-1.3.md)). To upgrade, see
> [Upgrading from 1.3](concepts/migration-guide.md).

## Start Here

| Goal | Documentation |
| --- | --- |
| Understand Assets vs SubAssets, and how telemetry identifies its source | [Assets & Sub-Assets](concepts/assets-and-sub-assets.md) |
| Build multi-step automations that the platform runs for you | [Applications & Skills](concepts/applications-and-skills.md) |
| Keep aircraft out of no-fly zones and bring them home safely on low battery | [No-fly zones and safe returns](concepts/airspace-safety.md) |
| Tune platform settings per organization or site, and decide which asset responds | [Technical configuration and dispatch rules](concepts/configuration.md) |
| Upgrade an existing 1.3.x installation or integration | [Upgrading from 1.3](concepts/migration-guide.md) |
| Use Zequent from a Java application | [Java Client SDK Quickstart](client-sdk/QUICKSTART.md) |
| Fly a waypoint mission from your own application | [Waypoint Missions](client-sdk/WAYPOINT_MISSIONS.md) |
| Start, stop and play back live video from an asset's camera | [Live Video Streams](client-sdk/LIVE_VIDEO.md) |
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
- **Applications & Skills** — compose commands, waits, conditions, human approvals and branches into versioned Skills and Applications. The platform runs them, picks the asset that responds and keeps flights out of no-fly zones; start them from your own code, on a schedule or from an event trigger. See [Applications & Skills](concepts/applications-and-skills.md).
- **Live telemetry & detections** — subscribe to real-time position, battery, and sensor telemetry, and AI detection results, streamed from every connected asset.
- **Live video** — start/stop live video streams from a connected asset's camera and view them in the Admin Console or your own player. See [Live Video Streams](client-sdk/LIVE_VIDEO.md).
- **Custom hardware integrations** — build a new edge adapter with the Edge SDK for any device that isn't already supported, using the same command/telemetry contract every built-in adapter uses.

## Customer Applications

Customer applications normally use the Client SDK and connect to the platform service endpoints exposed by your deployment. Every call carries a client credential (`ZQNT_CLIENT_TOKEN`), created in the Admin Console under **Manage → Access & Integrations → Credentials** — see [Client SDK Configuration](client-sdk/CONFIGURATION.md).

| SDK | Main docs |
| --- | --- |
| Java Client SDK | [Quickstart](client-sdk/QUICKSTART.md), [Configuration](client-sdk/CONFIGURATION.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE.md) |
| Python Client SDK | [Quickstart](client-sdk/QUICKSTART_PYTHON.md), [Configuration](client-sdk/CONFIGURATION_PYTHON.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE_PYTHON.md) |
| Go Client SDK | [Quickstart](client-sdk/QUICKSTART_GO.md), [Connector](client-sdk/CONNECTOR_GO.md) — no built-in retry/circuit-breaker/Stork layer, see the quickstart's "What this SDK deliberately does not do" |

## Platform Service Images

Zequent platform services are run from published container images. Use versioned tags for production deployments.

| Component | Image | Default port | Customer-facing purpose |
| --- | --- | ---: | --- |
| Connector Service | `ghcr.io/zequent/connector-service:2.0.0` | `8010` | System of record: assets, organizations, users, Applications and executions, schedulers, technical config, no-fly zones, telemetry persistence |
| Remote Control Service | `ghcr.io/zequent/remote-control-service:2.0.0` | `8002` | Direct asset commands such as takeoff, go-to, return-to-home, dock, camera, and manual-control commands |
| Live Data Service | `ghcr.io/zequent/live-data-service:2.0.0` | `8003` | Live telemetry, detections, and notification streams |
| Mission Autonomy Service | `ghcr.io/zequent/mission-autonomy-service:2.0.0` | `8004` | Runs Skill and Application executions: picks the asset that responds, plans routes around no-fly zones, returns aircraft home on low battery, handles human approvals; manages schedulers |
| Admin Console API | `ghcr.io/zequent/admin-console-service:2.0.0` | `8005` | HTTP/WebSocket API for the Admin Console: sign-in (including SSO), licensing, live streams, user management |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v2.0.0` | `3001` | Browser UI: monitoring, remote and manual control, live video, the Skill and Application editors, schedules and event triggers |

The Admin Console UI's image is `zqnt-platform-console`, not `zqnt-admin-console-dashboard` — that name is not a real published package, and pulling it fails. Its tags carry a `v` prefix (`v2.0.0`), unlike the core service images (`2.0.0`).

Platform services also require **Postgres (TimescaleDB)** and **Redis** — see [docker-compose.customer.yml](docker-compose.customer.yml). A runnable copy of the same file also lives at `core/docker-compose.customer.yml`, alongside `docker-compose.local.yml`/`docker-compose.env.yml`, for working directly in the monorepo — keep both copies in sync if you edit one.

## Edge Adapter Images

Use these adapter images when you want a ready-made integration. Use the Edge SDK when you need to build a custom adapter.

| Adapter | Image | Status | Notes |
| --- | --- | --- | --- |
| DJI | `ghcr.io/zequent/zqnt-edge-adapter-dji:2.0.0` | Available | DJI dock/drone integration. [Deployment guide](edge-sdk/edge-sdk-dji-adapter-deployment.md) |
| MAVLink | `ghcr.io/zequent/zqnt-adapter-mavlink:2.0.0` | Available | PX4/ArduPilot vehicles via MAVSDK. [Deployment guide](edge-sdk/edge-sdk-mavlink-adapter-deployment.md) |
| Sapient | `ghcr.io/zequent/zqnt-adapter-sapient:2.0.0` | Available | Bridges TCP SAPIENT edge nodes to gRPC. [Deployment guide](edge-sdk/edge-sdk-sapient-adapter-deployment.md) |
| RNS | `ghcr.io/zequent/zqnt-adapter-rns:2.0.0` | Available | Early-stage — implements asset registration and vendor custom commands only. [Deployment guide](edge-sdk/edge-sdk-rns-adapter-deployment.md) |
| Betaflight | `ghcr.io/zequent/zqnt-adapter-betaflight:2.0.0` | Available | Serial/USB flight-controller integration. [Deployment guide](edge-sdk/edge-sdk-betaflight-adapter-deployment.md) |
| AI Adapter | `ghcr.io/zequent/zqnt-adapter-ai:2.0.0` | Available | RTMP/RTSP video → YOLO detection → georeferenced results, with optional gimbal re-aim. Uses the standard Edge SDK adapter pattern. [Deployment guide](edge-sdk/edge-sdk-ai-adapter-deployment.md) |

The DJI image is `zqnt-edge-adapter-dji`, not `dji-adapter` — that older package name still exists but is stale/abandoned (only a floating `latest`, no versioned releases).

### Load-test / fleet simulator

`ghcr.io/zequent/zqnt-simulator:2.0.0` — a Go tool that simulates a fleet of edge adapters against
the live stack, for load-testing the Live Data gRPC telemetry ingest path and exercising
remote-control/admin-console fleet flows without real hardware. Optional, enabled via the
`simulator` Compose profile — not part of a normal customer deployment. See the repo's own
`simulator/README.md` for the full env var reference.

## Deployment Configuration

Container deployments use one deployment-local `.env` file referenced by [docker-compose.customer.yml](docker-compose.customer.yml). Start from [.env.customer.example](.env.customer.example) — copy it to `.env` next to the compose file and fill in every `<PLACEHOLDER>` (database/Redis passwords, your Ed25519 signing key, the licensing installation settings, public dashboard URLs) before starting the stack. It contains no Zequent-internal credentials — every value is either a safe structural default or a placeholder only you can fill in.

```yaml
services:
  connector-service:
    image: ghcr.io/zequent/connector-service:2.0.0
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

### Keys and credentials to create before the first start (and when upgrading from 1.3)

1. **Database and Redis passwords** — `POSTGRES_PASSWORD`/`DATABASE_PASSWORD` (an existing volume keeps the password it was created with) and `REDIS_PASSWORD`.
2. **User-token key pair** — `AUTH_PRIVATE_KEY`/`AUTH_PUBLIC_KEY`, plus `EXPORT_SIGNING_*` and `EXPORT_PLATFORM_KEK`.
3. **Service key pair** — `SERVICE_AUTH_PRIVATE_KEY`/`SERVICE_AUTH_PUBLIC_KEY`, a second Ed25519 pair. The compose file refuses to start without it. Keep the private key in your secret manager: edge credentials are minted from it.
4. **Edge credentials** — one per adapter and for the simulator (`<NAME>_EDGE_TOKEN` in `.env.customer`), issued after the platform is up in the Admin Console (Edge credentials) or offline with `core/scripts/mint-edge-credential.py`. The compose file passes the service public key to every adapter as `ZQNT_PLATFORM_PUBLIC_KEY` and keeps the platform's private keys and passwords away from them.
5. **Stream-auth hook secret** — `LIVE_STREAM_AUTH_HOOK_SECRET`, the same value in your media server's `authHTTPAddress` (only with a media server that authenticates).
6. **Stream-key pepper** — `LIVE_STREAM_KEYS_PEPPER`, optional, set once and never changed.

`LICENSING_PUBLIC_KEY` (needed on 1.3) is no longer needed: the services trust the Zequent license hub's key compiled into them.

## Licensing

Every platform service enforces an activated license before it will perform protected operations — a fresh deployment with no license activated will reject most requests. Licenses are organization- and seat-based: one license covers one organization, and each platform user you create consumes one of that organization's seats.

1. Configure the installation once: a `LICENSING_INSTALLATION_ID` generated once and `EXPORT_PLATFORM_KEK` (the license hub's public key is compiled into the services) (see [Client SDK Configuration](client-sdk/CONFIGURATION.md)). The platform may start with no licensed organization; the `system_admin` can still sign in.
2. Zequent issues a license for each of your organizations, by name. In the Admin Console, as `system_admin`: **Organizations → Create from license**, then paste the license key or drop the organization's `.zqnt` file. The organization is created with the id and name the license names, and its license is active immediately — no restart. For an organization that already exists, use **Activate license** on it (an organization admin can do this for their own organization on the License screen).
3. Services automatically refresh every organization's license lease afterward, across restarts — no further manual steps.

The license names the organization only once, when it is created; afterwards the organization's name is yours to change in the console.

See [Client SDK Configuration](client-sdk/CONFIGURATION.md) for the `LICENSING_*` environment variables.

## Admin Console

The Admin Console is split into an API image and a UI image.

| Component | Default local URL |
| --- | --- |
| Admin Console UI | `http://localhost:3001` |
| Admin Console API | `http://localhost:8005` |

The Admin Console provides browser workflows for asset monitoring, telemetry, remote and manual control, live streams, building Skills and Applications, schedules and event triggers, adapter management, users and licensing, and service health.


## SDK Requirements

| SDK | Requirements |
| --- | --- |
| Java Client SDK / Java Edge SDK | Java 25, Maven 3.9+ recommended, Quarkus 3.x for Quarkus applications |
| Python Client SDK / Python Edge SDK | Python 3.12+, `uv` recommended, `grpc.aio` |
| Go Client SDK / Go Edge SDK | Go 1.26+ (client) / Go 1.25+ (edge). Both modules are private and versioned as `/v2` (`github.com/Zequent/zqnt-client-sdk-go/v2`, `github.com/Zequent/zqnt-edge-sdk-go/v2`) — without `/v2` in the path Go installs the 1.3 line. See each quickstart's `go env -w GONOSUMDB`/`GONOPROXY` step. The Go Edge SDK is on an older API surface than the Java/Python Edge SDKs — see its [overview](edge-sdk/edge-sdk-go-overview.md). |

## Package Access

If you consume private Zequent packages, configure access to the relevant package registry before building your customer application or adapter.

For Maven packages, configure your `~/.m2/settings.xml` with a token that has package read access. The Python SDKs are not published on PyPI yet: install them from their private Git repositories at the release tag (`v2.0.0`), with a token that has read access to them.

## Production Notes

- Run platform services and provided adapters from published container images.
- Use versioned image tags for production, not `:latest`.
- Keep secrets outside Git.
- Use TLS and deployment-managed secrets for production environments.
- Use the Client SDKs for customer applications and the Edge SDKs for custom adapters.
