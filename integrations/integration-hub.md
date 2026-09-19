# Integration Hub -- Deployment Guide

Integration Hub is a general-purpose data bridge: it reads from a **source** (OPC-UA, Kafka, WebSocket,
OpenAPI/HTTP, Iosys HTTP), applies a configurable field mapping (with optional JavaScript
transforms), and writes to a **sink** (MQTT, Kafka, or — when connected to the Zequent platform, see
below — a running Skill execution). It's a separate service from the platform's own five core
services and runs standalone by default. It keeps its connector/mapping/vault configuration in its
own Postgres *schema* either way — a dedicated database for a standalone deployment, or (see
[docker-compose.local.yml](#docker-composelocalyml) below) the platform's own `zequent_db`
instance, own schema, for a console-embedded one. Either way its tables never mix with
`zequent_db`'s own asset/telemetry schema.

Repository: `zqnt-integration-hub` (Go backend + Next.js frontend). The **backend** is still its
own container image — see below. The **frontend** is natively embedded into the Admin Console
dashboard's own build now (`zqnt-console-dashboard/src/app/deploy/integrations`,
`src/features/integrations`) rather than deployed as a separate reverse-proxied container — see
[Console integration](#console-integration). Its source under `zqnt-integration-hub/frontend`
still exists and still builds standalone (its own Dockerfile, its own image) for a deployment that
wants Integration Hub's UI with no Zequent console at all — it's just no longer part of this
platform's own `docker-compose.local.yml` stack.

---

## Prerequisites

- Access to the Zequent container registry (`ghcr.io/zequent`)
- A Postgres instance for Integration Hub's own connector/mapping/vault configuration — either a
  dedicated database (standalone deployment) or `zequent_db` with `DATABASE_SCHEMA` set (see
  [docker-compose.local.yml](#docker-composelocalyml) below); either way it's a schema of its own,
  never mixed with `zequent_db`'s own asset/telemetry tables
- Optionally: the platform's Connector Service and Mission Autonomy Service, if you want configured
  connectors to appear as Skill-invocable capabilities (see [Skills & Capabilities
  integration](#skills--capabilities-integration) below) — **this specific integration depends on
  the platform's unreleased Skill/capability-execution model (Beta, not on the current 1.3.x
  release); see that section's own note before enabling `ZQNT_PLATFORM_ENABLED`**

---

## Container Images

| Component | Image | Default port | Purpose |
| --- | --- | ---: | --- |
| Integration Hub Backend | `ghcr.io/zequent/zqnt-integration-hub-backend:latest` | `8080` | Go API: connector/mapping/vault CRUD, the running bridge engine, and (optional) the Zequent platform bridge |

The console dashboard's own image (`ghcr.io/zequent/zqnt-admin-console-dashboard`) is what serves
Integration Hub's UI now — no separate frontend image is part of a console-embedded deployment.

`ghcr.io/zequent/zqnt-integration-hub-frontend` still exists as its own published image (built from
`zqnt-integration-hub/frontend`, `basePath=/integrations` baked in by default — see that repo's own
`next.config.ts`/`Dockerfile`) for a **standalone** deployment with no Zequent console involved at
all. It plays no role in a console-embedded one.

---

## Environment Variables

### Backend -- Required

| Variable | Description |
| --- | --- |
| `DATABASE_URL` | Postgres connection string, e.g. `postgres://user:pass@host:5432/dbname` — a dedicated database (standalone) or the platform's shared `zequent_db` (console-embedded, paired with `DATABASE_SCHEMA` below) |
| `VAULT_ENCRYPTION_KEY` | Symmetric key encrypting vault-stored connector credentials (usernames/passwords/certificates) at rest |

### Backend -- Optional

| Variable | Default | Description |
| --- | --- | --- |
| `DATABASE_SCHEMA` | unset (ordinary `public` schema) | Runs every query — this backend's own and, transparently, `golang-migrate`'s — against this Postgres *schema* instead of `public`, via a `search_path` connection option (see `db/schema.go`). Created automatically if missing. This is what lets `DATABASE_URL` point at a Postgres instance shared with other services (e.g. `zequent_db`) without table collisions; leave unset for a dedicated database, where `public` is already exclusively this backend's |
| `PORT` | `8080` | HTTP API port |
| `JAVASCRIPT_RUNNER_URL` | `http://localhost:8091` | Sandboxed JS execution service for mapping-step JavaScript transforms (see the repo's own `backend/js-runner`) |
| `PENDING_MESSAGE_STORE` | `memory` | Store-and-forward backend for failed sink writes: `memory` or `rabbitmq` |
| `RABBITMQ_URL`, `RABBITMQ_QUEUE`, `RABBITMQ_VHOST`, `RABBITMQ_USER`, `RABBITMQ_PASS` | -- | Only used when `PENDING_MESSAGE_STORE=rabbitmq` |

### Backend -- Zequent Platform Integration (optional, Beta)

> **This entire integration depends on the platform's unreleased Skill/capability-execution model**
> (Application → Skill → SkillExecution → SkillContract), which lives on unmerged `refactoring/*-v2`
> branches across the platform's services and protocol definitions — not on the current 1.3.x
> release. See [Skills & Capabilities Integration](#skills--capabilities-integration) below. Leave
> `ZQNT_PLATFORM_ENABLED` unset/`false` against a 1.3.x deployment.

Unset or `ZQNT_PLATFORM_ENABLED=false` (the default) runs Integration Hub fully standalone, with no
dependency on any Zequent platform service. Set `ZQNT_PLATFORM_ENABLED=true` to enable the
integration described below.

| Variable | Default | Description |
| --- | --- | --- |
| `ZQNT_PLATFORM_ENABLED` | `false` | Master switch for the platform bridge |
| `ZQNT_CONNECTOR_HOST` / `ZQNT_CONNECTOR_PORT` | `connector-service` / `8010` | Connector Service gRPC endpoint |
| `ZQNT_MISSION_AUTONOMY_HOST` / `ZQNT_MISSION_AUTONOMY_PORT` | `mission-autonomy-service` / `8004` | Mission Autonomy Service gRPC endpoint |
| `ZQNT_PLATFORM_LISTEN_ADDR` | `:9095` | Address this backend's own inbound gRPC server (for `SendCustomCommand`) binds to |
| `ZQNT_PLATFORM_ADVERTISED_ENDPOINT` | `integration-hub-backend:9095` | What other platform services should dial to reach the above — matters if you rename the container or run behind a different hostname |
| `ZQNT_ASSET_SN` | `integration-hub` | The serial number Integration Hub registers itself under as its own logical asset |
| `ZQNT_ASSET_NAME` | `Integration Hub` | Display name for that asset |
| `ZQNT_REDIS_URL` | `redis://localhost:6379` | Same Redis instance the platform's Java services use — required for `SendCustomCommand` dispatch to actually reach this backend (see [Skills & Capabilities integration](#skills--capabilities-integration)'s discovery-registration note) |

### Backend -- Auth (optional)

Unset or `ZQNT_AUTH_ENABLED=false` (the default) runs the API fully unauthenticated — the
standalone/customer-marketable shape. Set `ZQNT_AUTH_ENABLED=true` to require a valid platform
session token on every `/api/v1/*` call (this is what a console-embedded deployment should do —
see [Console integration](#console-integration)).

| Variable | Default | Description |
| --- | --- | --- |
| `ZQNT_AUTH_ENABLED` | `false` | Master switch for bearer-token auth on `/api/v1/*` |
| `AUTH_PUBLIC_KEY` | -- (required if enabled) | Base64 X.509 SubjectPublicKeyInfo, Ed25519 — the **same** value every core/ Java service's `AUTH_PUBLIC_KEY` already uses. Public key only: this backend can verify a token, never mint one. |
| `AUTH_EXPECTED_ISSUER` | unset (issuer read but not checked) | Optional `iss` claim allowlist |

Verification is a from-scratch EdDSA/Ed25519 check (`backend/auth`), not a JWT library — same wire
format and algorithm as `com.zqnt.utils.auth.PlatformTokenVerifier` (Java), deliberately kept
byte-for-byte compatible so a token admin-console issues verifies identically here.

### Frontend (standalone deployment only)

Only relevant when running `zqnt-integration-hub-frontend`'s own image standalone (no console) —
see [Container Images](#container-images). A console-embedded deployment doesn't use this at all.

| Variable | Default | Description |
| --- | --- | --- |
| `BACKEND_URL` | `http://localhost:8080` | Upstream for the frontend's own `/api/*` rewrite to the backend |
| `NEXT_PUBLIC_BASE_PATH` | `/integrations` (baked in at image build) | Path prefix every asset/route/API call this app makes is served under |

---

## docker-compose.local.yml

The platform's own local dev stack (`core/docker-compose.local.yml`) wires Integration Hub in
sharing `zequent_db` — the same Postgres/TimescaleDB instance every other core/ service already
uses — via `DATABASE_SCHEMA=schema_integration` rather than running a second Postgres container
just for this one service; `ZQNT_PLATFORM_ENABLED` is left at its default `false`-equivalent
(opt-in via the `ZQNT_PLATFORM_ENABLED` shell/`.env` variable). This also brings in a `js-runner`
service (built from `zqnt-integration-hub/backend/js-runner`, a sibling checkout — the build
context is a relative path across repos, so it only resolves if both are checked out next to each
other) — without one, `JAVASCRIPT_RUNNER_URL` points nowhere and mapping-step JavaScript
transforms fail. There is no `integration-hub-frontend` service in this file — the UI ships inside
`zqnt-console`'s own image (see [Console integration](#console-integration)). Copy that file's
`js-runner` / `integration-hub-backend` services as the starting point for your own deployment; add
`zqnt-integration-hub-frontend` yourself only if you want it running standalone alongside (unusual
— normally either console-embedded or standalone, not both).

A genuinely standalone deployment (no platform at all) still wants its own dedicated Postgres, not
`zequent_db` — see `zqnt-integration-hub`'s own `backend/docker-compose.yml` for that shape
(includes its own Postgres, RabbitMQ, and Kafka/MQTT test fixtures for exercising connector types
locally, none of which are part of this platform's own compose file).

---

## Skills & Capabilities Integration

> **Beta — depends on unreleased platform functionality.** Everything in this section requires the
> platform's Application → Skill → SkillExecution → SkillContract model, which currently lives only
> on unmerged `refactoring/*-v2` branches (protocol definitions, Connector/Mission Autonomy
> services, and the Admin Console's Skill graph editor) — not on `main`/the current 1.3.x release.
> `ObserveSkillContract` and the rest of the Skill Registry API described below do not exist on the
> platform you'd deploy today. Leave `ZQNT_PLATFORM_ENABLED=false` against a 1.3.x platform; this
> section documents the integration for when that model ships. See
> [Applications & Skills](../concepts/applications-and-skills-2.0.md) for the platform-side model this
> depends on.

When `ZQNT_PLATFORM_ENABLED=true`, Integration Hub's backend does three things on startup (and
again whenever a sink-role connector is created or updated, so this stays current without a
restart):

1. **Registers itself as its own logical asset** (serial number `ZQNT_ASSET_SN`) via Connector
   Service's `RegisterAsset` — the same mechanism every edge adapter uses.
2. **Mirrors every sink-role connector into the Skill Registry** via Connector Service's
   `ObserveSkillContract` RPC (`command_id` = `integration-hub.<connectorID>`) — this is what makes a configured sink
   selectable as an execution step in the Admin Console's Skill graph editor, the same way a DJI
   dock's or a mavlink drone's custom commands are.
3. **Runs a small inbound gRPC server implementing the platform's standard `EdgeAdapterService`
   contract**, the same one every edge adapter implements — but only `SendCustomCommand` is real;
   every other method (takeoff, go-to, manual control, ...) returns `NOT_IMPLEMENTED`, since
   Integration Hub doesn't drive hardware. When a Skill execution step invokes
   `integration-hub.<connectorID>`, this server resolves the connector ID, builds that connector's
   configured Sink, and writes the step's parameters through it.

This means: **a Skill can trigger any MQTT or Kafka sink you've configured in Integration Hub as
one of its execution steps**, using the exact same "invoke a capability" mechanism it already uses
for edge-adapter custom commands — no new platform-side plumbing was needed to support this.

### The reverse direction: a Source triggering a Skill

Integration Hub also has a fourth, purpose-built connector type for the opposite direction — a
**source** driving a running Skill forward:

- **Connector type `zqnt-skill`** (sink role): calls Mission Autonomy's `SignalSkillExecution` for
  every payload it's written, satisfying an `EVENT_WAIT` node in a running Skill execution. Because
  it's an ordinary sink, **any existing source type can drive it with zero special handling** — the
  normal read → map → write loop is all this needs. Configure it like any other sink:

  | Field | Description |
  | --- | --- |
  | `executionId` | The target Skill execution's ID. Supports `{fieldName}` placeholders resolved from the source payload — e.g. `{orderId}` if the execution ID itself comes from the data you're bridging. |
  | `nodeId` (optional) | The specific `EVENT_WAIT` node within the graph to signal. Also supports `{fieldName}` placeholders. |
  | `eventType` (optional) | A free-form event-type string, if the target node discriminates on one. |

  Example: an OPC-UA source watching a physical sensor, mapped so its reading becomes `{value}`,
  writing to a `zqnt-skill` sink with `executionId = "{skillExecutionId}"` — every sensor reading
  advances whichever Skill execution the payload identifies.

### Starting an Application from a bridge

A bridge can also start an Application through an **Event Trigger** of type `INTEGRATION`. The trigger names the bridge (`bridgeId`), a condition over the bridge's mapped payload (`telemetry_field`, `comparison_operator`, `comparison_value`), the target Application and Skill, and a cooldown. Wiring it this way keeps the target, condition and cooldown in one place that can be edited, instead of an `applicationId`/`skillId` inside the bridge's connector JSON. The trigger does not need to name an asset: when it fires with none, the platform picks one by policy (see [Which asset an execution runs on](../concepts/applications-and-skills-2.0.md#which-asset-an-execution-runs-on)), so a fire panel can report a zone and the nearest capable drone answers. See [Event triggers](../concepts/applications-and-skills-2.0.md#event-triggers).

### How a Skill actually reaches this backend

`SendCustomCommand` dispatch (Skill execution → this backend's inbound gRPC server) goes through
mission-autonomy's `GrpcEndpointRouter`, which resolves an asset SN to a gRPC endpoint via two
Redis lookups: `zqnt:edge-vendor:{sn}` (SN → vendor) and `zqnt:edge-endpoints:{vendor}` (vendor →
`{endpoint, online}`). This backend writes both directly on startup (`platform/redis_registration.go`,
using the `ZQNT_REDIS_URL` above) — registered under the `ASSET_VENDOR_ZQNT` value added to
`AssetVendor` in `asset.proto` specifically for non-hardware bridge assets like this one (every
other member names a real hardware vendor). Without this Redis registration a Skill can list this
backend's mirrored SkillContracts but never actually invoke one — found live-testing, not something
this doc is speculating about.

### Known limitations (current state)

- Asset/SkillContract registration retries only on the next full restart if Connector Service is
  unreachable at boot — there's no background retry loop yet.
- No theft/collision handling beyond what Connector Service's own SkillContract versioning already
  does.

---

## Console Integration

Integration Hub's UI is natively embedded in the Admin Console dashboard's own Next.js app —
`zqnt-console-dashboard/src/app/deploy/integrations/*` (pages) and
`src/features/integrations/*` (components, copied from `zqnt-integration-hub/frontend` and restyled
to import through the console's own generated API client — see `orval.config.ts`'s `integrationHub`
entry and `src/api/integrations-axios.ts`). It is **not** a separate reverse-proxied app anymore —
only its API is: `next.config.js` rewrites `/integrations/api/:path*` to
`INTEGRATION_HUB_BACKEND_ORIGIN` (default `http://localhost:8080`; `docker-compose.local.yml`
points it at `integration-hub-backend:8080`), same-origin from the browser.

Because the pages live under `/deploy/integrations/*`, they render inside the console's existing
`/deploy` shell and automatically inherit its `RequireAuth` session gate — no separate Integration
Hub login exists or is needed. The browser's existing console access token also rides along on
every `/integrations/api/*` call automatically (same shared axios instance/interceptor every other
console API call uses), so enabling this backend's own `ZQNT_AUTH_ENABLED` (see
[Backend -- Auth](#backend--auth-optional) above) actually protects the API too, not just the page.

To regenerate the API client after a real change to the backend's routes: re-copy
`zqnt-integration-hub/backend/docs/swagger.json` to `zqnt-console-dashboard/openapi/integration-hub.swagger.json`,
then run `pnpm orval` in `zqnt-console-dashboard`.
