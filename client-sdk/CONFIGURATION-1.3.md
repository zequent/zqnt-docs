# Zequent Client SDK - Configuration

This page describes the public configuration model for external developers and customer deployments.

Zequent platform services are run from published container images. Customer applications use the Client SDKs and connect to the service endpoints exposed by the deployment.

## Container Configuration

The public customer Compose template is [docker-compose.customer.yml](../docker-compose.customer.yml). It references one deployment-local `.env` file — start from [.env.customer.example](../.env.customer.example) (`cp .env.customer.example .env`, then fill in every `<PLACEHOLDER>`; a runnable copy of the compose file, and an actual `.env.customer` rather than the `.example` template, also lives at the same paths under `core/` for working directly in the monorepo).

```yaml
services:
  connector-service:
    image: ghcr.io/zequent/connector-service:1.3.2
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
| Connector Service | `ghcr.io/zequent/connector-service:1.3.2` | `8010` |
| Remote Control Service | `ghcr.io/zequent/remote-control-service:1.3.2` | `8002` |
| Live Data Service | `ghcr.io/zequent/live-data-service:1.3.2` | `8003` |
| Mission Autonomy Service | `ghcr.io/zequent/mission-autonomy-service:1.3.2` | `8004` |
| Admin Console API | `ghcr.io/zequent/admin-console-service:1.3.2` | `8005` |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v1.3.3` | `3001` |

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
| `LICENSING_INSTALLATION_ID` | All services | This installation's own id. Generate it once (e.g. `uuidgen`) and never change it: every organization's license activation is bound to it. The Helm chart generates and keeps it for you |
| `LICENSING_PUBLIC_KEY` | 1.3.x images only | 2.0 services trust only the Zequent license hub's key compiled into them and ignore this variable (with a warning). Set it from the installation bundle only while you run 1.3.x images |
| `LICENSING_SIGNING_KEY_ID` | 1.3.x images only | Key ID from the installation bundle, alongside `LICENSING_PUBLIC_KEY` |
| `LICENSING_MANUAL_REFRESH_INTERVAL` | Admin Console only | Optional. How often one organization may refresh its license by hand (default `30s`) |
| `LICENSE_SERVER_URL` | Admin Console only | Defaults to `https://api.zequent.com`; override only for a self-hosted/offline license server |
| `EXPORT_PLATFORM_KEK` | Admin Console only | 32 random bytes, base64 (`openssl rand -base64 32`). Seals every organization's license activation before it is stored; without it activations are lost on restart |
| `LICENSING_LICENSE_KEY` | Admin Console only | Optional. Only a bootstrap for the default organization; leave unset and license organizations in the console |

These are configured once per installation. Organizations are licensed afterwards, from the Admin Console, with no restart — see [Licensing](../README.md#licensing).

### Authentication

Every bearer token this deployment issues is signed with an Ed25519 keypair you generate yourself —
never reuse a value from any example or dev environment:

```bash
openssl genpkey -algorithm ed25519 -out auth-private.pem
openssl pkey -in auth-private.pem -pubout -out auth-public.pem
# base64 the PKCS8/SubjectPublicKeyInfo DER (or PEM, stripped of headers/newlines)
# into the two variables below
```

| Variable | Notes |
| --- | --- |
| `AUTH_PRIVATE_KEY` | Base64 PKCS8-encoded Ed25519 private key |
| `AUTH_PUBLIC_KEY` | Base64 SubjectPublicKeyInfo-encoded Ed25519 public key |
| `AUTH_ISSUER` | Token issuer — typically `<your-company>-admin-console` |
| `AUTH_EXPECTED_ISSUER` | Must match `AUTH_ISSUER` |
| `AUTH_SYSTEM_ADMIN_EMAIL` | Email for the initial system admin account |
| `OIDC_REDIRECT_URI` | Only needed if an organization connects its own SSO/OIDC identity provider |

### Service identity

Every call between the platform services carries a short-lived service token. It is signed with a
second Ed25519 key pair, separate from `AUTH_*`, one per installation. A 2.0 service refuses to start
without it, and refuses a key committed to Zequent's repositories.

| Variable | Applies to | Notes |
| --- | --- | --- |
| `SERVICE_AUTH_PRIVATE_KEY` | All five platform services | Ed25519 private key (PKCS8 DER, base64, or PEM). `openssl genpkey -algorithm ed25519 -out service.pem`, then `openssl pkey -in service.pem -outform DER \| base64 -w0`. Never give it to an adapter |
| `SERVICE_AUTH_PUBLIC_KEY` | All five platform services | The matching public key (`openssl pkey -in service.pem -pubout -outform DER \| base64 -w0`). Adapters get it as `ZQNT_PLATFORM_PUBLIC_KEY` |
| `SERVICE_AUTH_TOKEN_TTL` | All five platform services | Optional, ISO-8601 lifetime of the service tokens (default `PT5M`) |

### Live video

| Variable | Applies to | Notes |
| --- | --- | --- |
| `LIVE_STREAM_AUTH_HOOK_SECRET` | Admin Console only | Shared secret the media server presents to the stream-auth hook (MediaMTX: `?secret=<value>` on its `authHTTPAddress`). Unset, the hook answers anyone. URL-safe, e.g. `openssl rand -hex 32` |
| `LIVE_STREAM_KEYS_PEPPER` | Admin Console only | Optional pepper for operator-issued stream keys. Set once: changing or removing it invalidates every key issued under it |

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

### Client credential (`ZQNT_CLIENT_TOKEN`)

The platform refuses every gRPC call that carries no credential. An organization administrator issues a
**client credential** in the Admin Console under **Manage → Access & Integrations → Credentials** (kind
*Client application*); the token is shown once. Every client SDK sends it as `authorization: Bearer <token>`:

| SDK | How to pass it |
| --- | --- |
| Java | `ZequentClient.builder().clientToken(token)`, `zequent.client-token` (Quarkus), or `ZQNT_CLIENT_TOKEN` |
| Python | `ZequentClient(..., client_token=token)` or `ZQNT_CLIENT_TOKEN` (also for `from_env()`) |
| Go | `grpc.NewClient(target, append(auth.DialOptions(token), ...)...)`; `""` reads `ZQNT_CLIENT_TOKEN` |
| client-sapient | `ZQNT_CLIENT_TOKEN` |

A client credential belongs to exactly one organization and reaches only that organization's assets, Applications
and runs: asset lookups and payloads, the Skill list, Applications and executions, telemetry/detection/notification
subscriptions (a `*` subscription carries only the organization's assets) and asset commands. Users, organizations,
identity providers, licenses, triggers, schedules, policies, technical configuration and asset registration are
never reachable with it. `UNAUTHENTICATED` means no, an expired or a revoked credential; `PERMISSION_DENIED` means
the call is outside that scope. Revoking it in the console takes effect on every service within seconds.

## Admin Console

The Admin Console has two images:

| Component | Image | Default local URL |
| --- | --- | --- |
| Admin Console API | `ghcr.io/zequent/admin-console-service:1.3.2` | `http://localhost:8005` |
| Admin Console UI | `ghcr.io/zequent/zqnt-platform-console:v1.3.3` | `http://localhost:3001` |

The Admin Console UI needs public API and WebSocket URLs that are reachable from the user's browser.

| Variable | Purpose |
| --- | --- |
| `BACKEND_ORIGIN` | Backend origin the UI container's own server-side proxy uses inside the deployment network. Silently defaults to `http://localhost:8005` if unset, which inside that container resolves to the UI container itself, not the Admin Console API — producing a UI that loads but has no working backend connection |
| `LICENSE_API_ORIGIN` | The UI container's server-side proxy target for license operations. Defaults to `http://localhost:4000` if unset — a local mock this repo doesn't run — so set it to `https://api.zequent.com` (or your self-hosted license server) explicitly |
| `ADMIN_CONSOLE_CORS_ORIGINS` | The public dashboard origin(s) the Admin Console API accepts requests from |
| `NEXT_PUBLIC_BACKEND_ORIGIN` | Public Admin Console API URL, general-purpose |
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
| `ZQNT_EDGE_TOKEN` | The adapter's edge credential. A 2.0 platform refuses its calls without one. Issued in the Admin Console (Access & Integrations → Credentials, kind *Edge adapter*) or offline with `core/scripts/mint-edge-credential.py` from `SERVICE_AUTH_PRIVATE_KEY`; the Integration Hub needs one of kind *Integration Hub* |
| `ZQNT_PLATFORM_PUBLIC_KEY` | The platform's `SERVICE_AUTH_PUBLIC_KEY`. The adapter refuses every command not signed with it |
| `ZQNT_EDGE_AUTH_DISABLED` | `true` accepts unsigned commands. Local test stacks only, never a deployment |
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
