# Zequent Client SDK - Configuration

This page describes the public configuration model for external developers and customer deployments.

Zequent platform services are run from published container images. Customer applications use the Client SDKs and connect to the service endpoints exposed by the deployment.

## Container Configuration

The public customer Compose template is [docker-compose.customer.yml](../docker-compose.customer.yml). It references one deployment-local `.env` file — start from [.env.customer](../.env.customer) (`cp .env.customer .env`, then fill in every `<PLACEHOLDER>`; a runnable copy of both files also lives at the same paths under `core/` for working directly in the monorepo).

```yaml
services:
  connector-service:
    image: ghcr.io/zequent/connector-service:1.3.1
    env_file:
      - .env
```

Use the same `env_file: .env` pattern for Zequent service images, Admin Console images, adapter images, and customer application containers.

Do not commit `.env` files. Store credentials and deployment-specific values in your deployment environment or secret manager.

Start the public template with:

```bash
docker compose -f docker-compose.customer.yml up -d
```

Optional adapter images are enabled through Compose profiles, for example:

```bash
docker compose -f docker-compose.customer.yml --profile edge-dji up -d
```

## Platform Service Images

| Component | Image | Default port |
| --- | --- | ---: |
| Connector Service | `ghcr.io/zequent/connector-service:1.3.1` | `8010` |
| Remote Control Service | `ghcr.io/zequent/remote-control-service:1.3.1` | `8002` |
| Live Data Service | `ghcr.io/zequent/live-data-service:1.3.1` | `8003` |
| Mission Autonomy Service | `ghcr.io/zequent/mission-autonomy-service:1.3.1` | `8004` |
| Admin Console API | `ghcr.io/zequent/admin-console-service:1.3.1` | `8005` |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v1.0.5` | `3001` |

Use versioned image tags for production deployments (as above — not `:latest`). The Admin Console UI's
version line is independent of the core services' `1.3.x` line.

## Adapter Images

| Adapter | Image | Status |
| --- | --- | --- |
| DJI | `ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0` | Available |
| MAVLink | `ghcr.io/zequent/zqnt-adapter-mavlink:1.3.0` | Available |
| Sapient | `ghcr.io/zequent/zqnt-adapter-sapient:1.3.0` | Available |
| RNS | `ghcr.io/zequent/zqnt-adapter-rns` — no versioned release yet, `latest` only | Source only |
| Betaflight | No published image yet | Source only |
| AI Adapter | No published image yet | Early access |

## Required `.env` Variables (Platform Deployment)

These go in the deployment-local `.env` file used by [docker-compose.customer.yml](../docker-compose.customer.yml) — not in your customer application's own `.env`.

### Database and cache

| Variable | Example | Notes |
| --- | --- | --- |
| `DATABASE_URL` | `jdbc:postgresql://postgres:5432/zequent_db` | JDBC URL, used by Hibernate |
| `DATABASE_REACTIVE_URL` | `postgresql://postgres:5432/zequent_db` | Reactive driver URL (no `jdbc:` prefix) |
| `DATABASE_USER` | `postgres` | |
| `DATABASE_PASSWORD` | — | Set your own; do not use the Postgres default in production |
| `REDIS_URL` | `redis://redis:6379` | |

### Licensing

Every platform service verifies a license lease before performing protected operations — see [Licensing](../README.md#licensing).

| Variable | Applies to | Notes |
| --- | --- | --- |
| `LICENSING_INSTALLATION_ID` | All services | A stable identifier for this deployment |
| `LICENSING_PUBLIC_KEY` | All services | Public key used to verify the license lease signature; provided with your license |
| `LICENSING_LICENSE_KEY` | Admin Console only | The license key issued to your organization |
| `LICENSE_SERVER_URL` | Admin Console only | Defaults to `https://api.zequent.com`; override only for a self-hosted/offline license server |

Activation is a one-time step performed from the Admin Console once it's running — see [Licensing](../README.md#licensing).

### Authentication & SSO

Local email+password authentication works with no configuration beyond the variables below. Every
platform user belongs to one organization by default; connecting an organization to its own OIDC
identity provider (SSO) instead is an opt-in step performed after the deployment is running — see
[Organizations, Users & Single Sign-On](../admin/organizations-and-sso.md) for the full walkthrough
(bootstrapping the first user, creating organizations, configuring SSO, and testing it locally).

| Variable | Applies to | Notes |
| --- | --- | --- |
| `AUTH_PRIVATE_KEY` | Admin Console only | Ed25519 private key (PKCS8, base64) that signs every bearer token this deployment issues. Generate your own for production — never reuse a development default |
| `AUTH_PUBLIC_KEY` | All services | The matching Ed25519 public key, used to verify bearer tokens. Same value on every service |
| `AUTH_ISSUER` | Admin Console only | Issuer claim this deployment stamps on tokens it mints |
| `AUTH_EXPECTED_ISSUER` | All services | Issuer claim every service requires on a token it verifies — normally the same value as `AUTH_ISSUER` |
| `AUTH_SYSTEM_ADMIN_EMAIL` | Connector Service | Email for the one `system_admin` user seeded automatically on first boot against an empty database |
| `OIDC_REDIRECT_URI` | Admin Console only | Only needed if any organization uses SSO — your Admin Console UI's own callback URL, registered as the allowed redirect URI on every configured identity provider |

## Client SDK Service Endpoints

Customer applications need the platform service hostnames and ports.

When the customer application runs inside the same Compose or Kubernetes network, use the service names:

| Variable | Typical value |
| --- | --- |
| `REMOTE_CONTROL_SERVICE_HOST` | `remote-control-service` |
| `REMOTE_CONTROL_SERVICE_PORT` | `8002` |
| `LIVE_DATA_SERVICE_HOST` | `live-data-service` |
| `LIVE_DATA_SERVICE_PORT` | `8003` |
| `MISSION_AUTONOMY_SERVICE_HOST` | `mission-autonomy-service` |
| `MISSION_AUTONOMY_SERVICE_PORT` | `8004` |
| `CONNECTOR_SERVICE_HOST` | `connector-service` |
| `CONNECTOR_SERVICE_PORT` | `8010` |

When the customer application runs on the host and connects to exposed local ports, use `localhost` for the host values.

## Admin Console

The Admin Console has two images:

| Component | Image | Default local URL |
| --- | --- | --- |
| Admin Console API | `ghcr.io/zequent/admin-console-service:1.3.1` | `http://localhost:8005` |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v1.0.5` | `http://localhost:3001` |

The Admin Console UI needs public API and WebSocket URLs that are reachable from the user's browser.

| Variable | Purpose |
| --- | --- |
| `BACKEND_ORIGIN` | Backend origin used by the UI container inside the deployment network |
| `NEXT_PUBLIC_BACKEND_AUTH_API_HOST` | Public Admin Console API URL |
| `NEXT_PUBLIC_BACKEND_OPERATION_API_HOST` | Public operation API URL |
| `NEXT_PUBLIC_BACKEND_SCHEDULER_API_HOST` | Public scheduler API URL |
| `NEXT_PUBLIC_BACKEND_ASSET_API_HOST` | Public asset API URL |
| `NEXT_PUBLIC_BACKEND_ORGANIZATION_API_HOST` | Public organization API URL |
| `NEXT_PUBLIC_BACKEND_ASSET_WS_HOST` | Public asset WebSocket URL |
| `NEXT_PUBLIC_BACKEND_DRC_WS_HOST` | Public direct remote control WebSocket URL |
| `NEXT_PUBLIC_BACKEND_NOTIFICATION_WS_HOST` | Public notification WebSocket URL |

For a local Compose deployment, these URLs normally point to `localhost:8005`.

## Edge Adapter Configuration

Ready-made adapter images and custom Edge SDK adapters need a reachable adapter endpoint plus any device-specific credentials.

| Variable | Purpose |
| --- | --- |
| `EDGE_ADAPTER_TARGET_ENDPOINTS` | Host and port where the platform can reach the adapter |
| Device/broker credentials | Credentials required by the selected adapter integration |
| Storage credentials | Optional credentials when the adapter uploads or downloads mission files/media |

The exact device-specific values depend on the selected adapter image.

## Deployment Notes

- Keep one `.env` per deployment environment.
- Do not publish `.env` files in public documentation or source repositories.
- Keep secrets in your orchestrator's secret mechanism for production.
- Use service names for container-to-container communication.
- Use public hostnames or exposed local ports for browser-facing URLs.
- Use the Client SDK for customer applications.
- Use the Edge SDK when you need to build a custom adapter.
