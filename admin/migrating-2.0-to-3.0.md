# Migrating an installation from 2.0 to 3.0 (preview)

> **Preview.** Zequent **3.0** is not released yet; this runbook describes the preview builds and
> will change before the release. Until then, run it only against a copy of your installation.

This is the operator's step-by-step guide for moving an existing **2.0** installation — the
customer Docker Compose setup or the Zequent Helm chart — to **3.0**. What changes and why, which
breaking changes affect integrations, and which parts must move together is explained in
[Upgrading the platform to 3.0](../concepts/upgrading-platform-to-3.0.md); read it first. OpenBao
and Keycloak themselves are described in [Secrets and sign-in](secrets-and-sign-in.md).

- [Overview](#overview)
- [1. Prerequisites](#1-prerequisites)
- [2. Back up](#2-back-up)
- [3. Set up Keycloak and OpenBao](#3-set-up-keycloak-and-openbao)
- [4. Change the configuration](#4-change-the-configuration)
- [5. Move users and sealed secrets](#5-move-users-and-sealed-secrets)
- [6. Upgrade the services](#6-upgrade-the-services)
- [7. What the database migrations do](#7-what-the-database-migrations-do)
- [8. Verify](#8-verify)
- [9. Roll back](#9-roll-back)
- [10. Adapters and client applications](#10-adapters-and-client-applications)

## Overview

| | 2.0 | 3.0 |
| --- | --- | --- |
| Sign-in | the platform's own password login (password hashes in Postgres), optional single sign-on per organization configured in the platform | **Keycloak** only; single sign-on is brokered by Keycloak |
| Secrets | environment variables and Kubernetes Secrets; license activations sealed with `EXPORT_PLATFORM_KEK` | **OpenBao**: KV for passwords and peppers, Transit keys for signing and sealing |
| Integration Hub credentials | encrypted in its database with `VAULT_ENCRYPTION_KEY` | OpenBao KV only |
| Infrastructure | Postgres/TimescaleDB, Redis | Postgres/TimescaleDB, Redis, **Keycloak** (with its own database), **OpenBao** |

The upgrade is a maintenance window: all core services and the console stop on 2.0 and start on
3.0 together. Plan for the database migration, which rewrites timestamp columns and can take minutes
on a database with a long history.

## 1. Prerequisites

| What | Why |
| --- | --- |
| The installation runs the latest 2.0.x release | 3.0's migrations start from the 2.0 database |
| A copy of the production database to rehearse on | the migration cannot be undone except by restoring a backup ([Roll back](#9-roll-back)) |
| Free disk space on Postgres of at least the size of the detections table | compressed detection chunks are decompressed for the migration and compressed again later |
| A DNS name and a TLS certificate for Keycloak (for example `auth.<your-domain>`) | people's browsers are sent to Keycloak to sign in; every token names it as issuer |
| Docker Compose 2.23 or newer (Compose) | the 3.0 compose file uses inline configs for OpenBao and Keycloak |
| Helm 3 and the 3.0 chart (Kubernetes) | `openbao.*` and `keycloak.*` values |
| Every device paired with a claim code, every human approval given by a signed-in user | 3.0 removes the legacy pairing paths and refuses approvals from credentials — see [Recommended upgrade order](../concepts/upgrading-platform-to-3.0.md#recommended-upgrade-order) |
| Copies of the 2.0 secrets: `.env.customer` or the Kubernetes Secrets, in particular `EXPORT_PLATFORM_KEK`, `VAULT_ENCRYPTION_KEY`, `SERVICE_AUTH_PRIVATE_KEY` / `SERVICE_AUTH_PUBLIC_KEY` and `LIVE_STREAM_KEYS_PEPPER` | needed to move sealed secrets, to keep 2.0 credentials valid and for a rollback |

## 2. Back up

Stop nothing yet; take the backups right before the maintenance window and again after the 2.0
services are stopped (step 6.1), so the second one has no writes missing.

### Postgres (with TimescaleDB)

The database `zequent_db` holds the platform's data and, in its own schema, the Integration Hub's.

```bash
# Docker Compose
docker compose exec -T postgres pg_dump -U postgres -Fc zequent_db > zequent_db-$(date +%F).dump

# Kubernetes: in the pod of your Postgres
kubectl -n <namespace> exec <postgres pod> -- pg_dump -U postgres -Fc zequent_db > zequent_db-$(date +%F).dump
```

`pg_dump` may warn about circular foreign keys in TimescaleDB's catalog; that is expected. A restore
needs the **same TimescaleDB version** and runs between `timescaledb_pre_restore()` and
`timescaledb_post_restore()` (see [Roll back](#9-roll-back)). Test the restore once into an empty
database before the window.

### Redis

Redis holds license leases, sessions, the simulator's fleet and, on installations that came from
older releases, no-fly zones the console kept only in Redis (connector imports them into Postgres,
3.0 still reads them from there). Take a snapshot:

```bash
# Docker Compose (the container has the password in REDISCLI_AUTH)
docker compose exec redis redis-cli BGSAVE
docker compose exec redis redis-cli LASTSAVE      # repeat until the timestamp changes
docker compose cp redis:/data ./redis-$(date +%F)
```

On Kubernetes, use your Redis installation's snapshot procedure (`BGSAVE`, then copy the RDB and
AOF files from its volume).

### OpenBao and Keycloak

Nothing to back up: 2.0 has neither. From the first 3.0 start on, back them up as described in
[Backup and restore](secrets-and-sign-in.md#backup-and-restore), and keep the OpenBao seal key apart
from every backup.

## 3. Set up Keycloak and OpenBao

Both run next to the 2.0 services without affecting them, so this step can be done before the
maintenance window.

### Keycloak

| | Docker Compose | Helm |
| --- | --- | --- |
| Switch on | `--profile keycloak` | `keycloak.enabled: true` |
| Public address | `KEYCLOAK_URL` (TLS reverse proxy in front of port `8180`) | `keycloak.ingress.host`, TLS Secret `keycloak.tls.secretName` |
| Console address (redirect target) | `ZQNT_CONSOLE_URL` | `console.frontend.ingress.host` |
| Database | database `keycloak` on the platform's Postgres, created by `keycloak-db-init` with `KEYCLOAK_DB_PASSWORD` | its own `keycloak-postgres` (`keycloak.database.*`), or an existing database |
| Secrets | `KEYCLOAK_DB_PASSWORD`, `KEYCLOAK_ADMIN_PASSWORD`, `KEYCLOAK_ADMIN_CONSOLE_CLIENT_SECRET` — each `openssl rand -hex 32` | generated once into `<release>-credentials` |

Keycloak imports the realm `zequent` (clients `zqnt-console` and `zqnt-admin-console`) on its first
start. Replace the temporary administrator `admin` with a permanent one afterwards. Details:
[How people sign in](secrets-and-sign-in.md#how-people-sign-in).

### OpenBao

| | Docker Compose | Helm |
| --- | --- | --- |
| Switch on | `--profile openbao` | `openbao.enabled: true` (`openbao.replicas` 1, or 3/5 for high availability) |
| Seal key | generated once into the volume `openbao_seal` | generated once into `<release>-credentials`, or `openbao.seal.existingSecret` |
| Mounts, Transit keys, policies, logins | `openbao-init`, on every start | Job `openbao-configure`, after every install and upgrade |
| Logins of the services | one AppRole per workload; role-id and secret-id in the volume `openbao_credentials` | Kubernetes auth, one role per workload, bound to its ServiceAccount |

Both create the Transit keys `service-token`, `credential`, `export-signing` and `platform-kek`
under `zqnt-transit/`, and write each KV secret **once**, when its path does not exist yet, from
the values you already have:

| 2.0 setting (`.env.customer` / chart Secret) | OpenBao path (mount `zqnt/`) | Field |
| --- | --- | --- |
| `POSTGRES_PASSWORD` / `DATABASE_PASSWORD` | `infrastructure/postgres` | `password` |
| `REDIS_PASSWORD` | `infrastructure/redis` | `password` |
| `LIVE_STREAM_AUTH_HOOK_SECRET` | `infrastructure/mediamtx` | `hook-secret` |
| object storage keys (Helm) | `infrastructure/object-storage` | `platform-`, `dock-`, `adapter-access-key` / `-secret-key` |
| DJI MQTT broker credentials | `infrastructure/mqtt` | `username`, `password`, `dock-username`, `dock-password` |
| `DJI_APP_KEY`, `DJI_APP_ID`, `DJI_APP_LICENSE` | `platform/edge-adapter-dji/cloud-api` | `app-key`, `app-id`, `app-license` |
| `LIVE_STREAM_KEYS_PEPPER` | `platform/peppers/stream-keys` | `value` |
| `ZQNT_PEPPER_WEBHOOK_TOKENS` (new) | `platform/peppers/webhook-tokens` | `value` |
| `ZQNT_PEPPER_CLAIM_CODES` (new) | `platform/peppers/claim-codes` | `value` |
| `ZQNT_PEPPER_AGENT_PSEUDONYMS` (new) | `platform/peppers/agent-pseudonyms` | `value` |
| `KEYCLOAK_ADMIN_CONSOLE_CLIENT_SECRET` | `platform/admin-console/keycloak` | `client-secret` |
| `GRAFANA_ADMIN_PASSWORD` (optional monitoring) | `infrastructure/grafana` | `admin-password` |

A pepper is the key under which tokens, codes and stream keys are stored as hashes. **If your 2.0
installation had `LIVE_STREAM_KEYS_PEPPER` set, keep exactly that value.** Generate the three new
peppers once (`openssl rand -hex 32`) and never change them: the first 3.0 start hashes every
existing webhook token and claim code under them. A pepper you do not set is generated by
`openbao-init` / the chart, which is fine for the new ones and wrong for an existing
`LIVE_STREAM_KEYS_PEPPER`.

After the first start, OpenBao is the source of these values: change one with
`bao kv patch -mount=zqnt <path> <field>=…` and restart the workloads that read it.

## 4. Change the configuration

### Docker Compose (`.env.customer` and `docker-compose.customer.yml`)

Take the 3.0 `docker-compose.customer.yml` and `.env.customer.example` and carry your values over.

| Change | Setting | What to do |
| --- | --- | --- |
| Removed | `EXPORT_PLATFORM_KEK` | No longer read: license activations and AI model keys are sealed with OpenBao's `platform-kek`. Keep the value in your backup until step 5 is done |
| Removed | `OIDC_REDIRECT_URI` | Single sign-on now goes through Keycloak; the redirect URI an organization registers with its identity provider is shown on the console's **Single Sign-On** screen |
| No longer read by the core services | `AUTH_PRIVATE_KEY`, `AUTH_ISSUER`, `AUTH_PUBLIC_KEY`, `AUTH_EXPECTED_ISSUER`, `AUTH_REFRESH_COOKIE_SECURE` | 3.0 issues no sign-in tokens of its own. The template still lists `AUTH_PUBLIC_KEY` / `AUTH_EXPECTED_ISSUER` for the simulator's control API |
| Now required | `LIVE_STREAM_KEYS_PEPPER` | Was optional. Keep your 2.0 value if you had one |
| Added, required | `ZQNT_PEPPER_WEBHOOK_TOKENS`, `ZQNT_PEPPER_CLAIM_CODES`, `ZQNT_PEPPER_AGENT_PSEUDONYMS` | `openssl rand -hex 32` each, once. The compose file refuses to start without them |
| Added (Keycloak) | `KEYCLOAK_URL`, `ZQNT_CONSOLE_URL`, `KEYCLOAK_DB_PASSWORD`, `KEYCLOAK_ADMIN_PASSWORD`, `KEYCLOAK_ADMIN_CONSOLE_CLIENT_SECRET` | see [step 3](#keycloak) |
| Added (optional) | `GRAFANA_ADMIN_PASSWORD`, `PROMETHEUS_PORT`, `GRAFANA_PORT` | only with `--profile monitoring` (needs `--profile openbao`) |
| Added, set by the compose file | `ZQNT_PLATFORM_KEYS_URL` for adapters and the simulator | the key set admin-console publishes; nothing to configure |
| Unchanged | `POSTGRES_PASSWORD`, `DATABASE_*`, `REDIS_*`, `SERVICE_AUTH_PRIVATE_KEY`, `SERVICE_AUTH_PUBLIC_KEY`, `LICENSING_*`, `LICENSE_SERVER_URL`, `AUTH_SYSTEM_ADMIN_EMAIL`, `ADMIN_CONSOLE_CORS_ORIGINS`, `NEXT_PUBLIC_*`, edge tokens | keep. Keep `SERVICE_AUTH_PUBLIC_KEY` in any case: edge and client credentials issued on 2.0 are checked against it |
| Image tags | every `image:` | the 3.0 tags from the release notes, for all core services and the console at once |

The 3.0 core services additionally need these settings (every core service unless noted):

| Setting | Value |
| --- | --- |
| `ZQNT_AUTH_KEYCLOAK_ISSUER` | `https://<keycloak>/realms/zequent` — a core service does not start without it |
| `ZQNT_AUTH_KEYCLOAK_JWKS_URL` | optional; Keycloak's keys from inside the network, for example `http://keycloak:8080/realms/zequent/protocol/openid-connect/certs` |
| `ZQNT_AUTH_KEYCLOAK_ADMIN_URL` (admin-console) | optional; for example `http://keycloak:8080` |
| `ZQNT_AUTH_KEYCLOAK_ADMIN_CLIENT_SECRET` (admin-console) | the `zqnt-admin-console` client secret, unless it is read from OpenBao |
| `ZQNT_SECRETS_PROVIDER` | `openbao` — required at least on admin-console: without it no license activation can be stored |
| `OPENBAO_ADDR` | `http://openbao:8200` |
| `OPENBAO_APPROLE_ROLE_ID_FILE`, `OPENBAO_APPROLE_SECRET_ID_FILE` | the files `role-id` and `secret-id` from the workload's folder in the volume `openbao_credentials` |

> **Open before release:** the preview `docker-compose.customer.yml` starts OpenBao and Keycloak but
> does not pass the settings above to the core services yet. Until it does, add them to the core
> services yourself, or run the upgrade on the Helm chart.

### Helm (`zqnt-platform-v2` values)

| Change | Value | What to do |
| --- | --- | --- |
| Removed | `auth.privateKeyKey` (`AUTH_PRIVATE_KEY`), `auth.issuer`, `auth.refreshCookieSecure` | 3.0 issues no sign-in tokens of its own |
| Removed | `auth.exportPlatformKekKey` (`EXPORT_PLATFORM_KEK`) | sealing moved to OpenBao Transit `platform-kek` |
| Removed | `integrationHub.vault.*` (`VAULT_ENCRYPTION_KEY`) | Hub credentials live in OpenBao; see the callout in [step 6](#6-upgrade-the-services) |
| Removed | `integrationHub.platform.assetSn`, `integrationHub.platform.assetName` | the Integration Hub is no asset any more |
| Removed | `services.djiAdapter.licensing.organizationId`, `services.simulator.organizationId` | the organization comes from the claim code or the edge credential |
| Kept | `auth.existingSecret` with `EXPORT_SIGNING_PRIVATE_KEY` / `EXPORT_SIGNING_PUBLIC_KEY`, `auth.publicKeyKey`, `auth.expectedIssuer`, `auth.systemAdminEmail`, `serviceAuth.*`, `licensing.*`, `database.*`, `redis.*` | with OpenBao, `export-signing` signs and the public half only verifies files exported before |
| Added, required | `keycloak.enabled: true`, `keycloak.ingress.host`, `keycloak.tls.secretName` | see [step 3](#keycloak) |
| Added, required | `openbao.enabled: true` | the Integration Hub and the sealing of license activations need it |
| Added | `services.connector.peppers.*`, `console.backend.peppers.*` | the new peppers; generated once into `<release>-credentials` unless you name an `existingSecret`. `console.backend.liveStream.keysPepper` stays as it is |
| Added | `services.connector.startupSeconds` (default `1800`) | time the connector may take for its migrations before liveness applies; raise it for a very large database |
| Added (optional) | `availability.*`, `autoscaling.*`, `services.<name>.autoscaling`, `console.backend.replicas`, `agentService.*`, `observability.grafana.keycloak.*`, `networkPolicy.kubeApiServer` | replicas and autoscaling, see [Sizing and capacity](sizing-and-capacity.md); the AI assistant; Grafana sign-in. Narrow `networkPolicy.kubeApiServer.cidr` to your API server when OpenBao is on |
| Image tags | `services.*.tag`, `console.*.image.tag`, `integrationHub.image.tag` | the 3.0 tags from the release notes |

### Default passwords are refused

A 3.0 service started outside development mode refuses to start when any of these holds a value
from Zequent's local development setup or an unfilled `<PLACEHOLDER>` of `.env.customer.example`:

| Checked | Since |
| --- | --- |
| Postgres password, Redis password, object storage secret key (also when read from OpenBao) | **new in 3.0** |
| The four peppers, the stream-auth hook secret, the `zqnt-admin-console` client secret, the OpenBao login | new in 3.0 |
| Service key pair, export signing pair, platform KEK, license public key | already in 2.0 |

The error names the setting, never the value. An installation that still runs on a password copied
from a developer setup must change it **before** the upgrade (for Postgres: `ALTER USER … PASSWORD`,
then the setting).

## 5. Move users and sealed secrets

The 3.0 connector refuses to migrate the database while anything that 2.0 kept in its own database
has not moved:

| Guard | Refuses while | What has to move |
| --- | --- | --- |
| Users and single sign-on | a user who signs in with a password has no Keycloak account, or an organization's single sign-on provider still has its client secret in the platform | every password user into Keycloak (account linked to the platform user, member of their organization, ideally with their existing password); every organization's single sign-on into Keycloak |
| Sealed secrets | a license activation (or an AI model key) is still sealed with `EXPORT_PLATFORM_KEK` | every organization's license activation, opened with `EXPORT_PLATFORM_KEK` and sealed again with OpenBao's Transit key `platform-kek` |

On a 2.0 database both guards apply: the seeded system administrator alone is a password user
without a Keycloak account, and every licensed organization has an activation sealed with
`EXPORT_PLATFORM_KEK`.

> **Open before release:** there is no supported way yet to move a 2.0 installation's password users
> into Keycloak and its license activations into OpenBao Transit. The preview builds that did this
> on their first start no longer do, and the tool that replaces them is not available yet. On a
> plain 2.0 database the 3.0 connector therefore stops at this point with
> `… password user(s) have no Keycloak account yet …` or
> `… license activation(s) … are still sealed with EXPORT_PLATFORM_KEK`, and the database is left
> half-migrated (restore the backup, see [Roll back](#9-roll-back)). This section will describe the
> supported steps before 3.0 is released. Do not upgrade a production installation until then.

The error messages name a way past each guard that gives the data up instead of moving it. It is
not a migration — it costs every password and every activation — and is only meant for test
installations; talk to Zequent before using it anywhere else:

- A user that is to lose its password: `UPDATE schema_zequent.users SET password_hash = NULL WHERE id = …`.
  After the upgrade admin-console creates their Keycloak account without a password; an
  administrator sets one in Keycloak, or the person uses *Forgot password?* once Keycloak can send
  email.
- An organization's single sign-on to configure again: `UPDATE schema_zequent.identity_providers SET client_secret = '' WHERE organization_id = …`,
  then set it up again on **Single Sign-On** after the upgrade
  ([Microsoft Entra ID for an organization](secrets-and-sign-in.md#microsoft-entra-id-for-an-organization)).
- A license activation given up: delete its row from `schema_zequent.license_activation_credentials`,
  then activate the organization's license again in the console (**Organizations → Activate license**).

## 6. Upgrade the services

### 6.1 Stop 2.0

Stop the five core services, the console and the Integration Hub. Keep Postgres, Redis, OpenBao and
Keycloak running. Take the second backup ([step 2](#2-back-up)).

### 6.2 Start 3.0

**Docker Compose**

```bash
docker compose --env-file .env.customer -f docker-compose.customer.yml \
  --profile openbao --profile keycloak pull
docker compose --env-file .env.customer -f docker-compose.customer.yml \
  --profile openbao --profile keycloak up -d openbao-seal-key openbao openbao-init keycloak-db-init keycloak
docker compose --env-file .env.customer -f docker-compose.customer.yml \
  --profile openbao --profile keycloak up -d connector-service
docker compose logs -f connector-service       # wait for the migrations to finish
docker compose --env-file .env.customer -f docker-compose.customer.yml \
  --profile openbao --profile keycloak up -d
```

**Helm**

One `helm upgrade` with the 3.0 chart, the changed values and the 3.0 image tags. OpenBao, its
configure Job, Keycloak and its realm Job come up with it; the core services restart a few times
until OpenBao is configured. Watch the connector:

```bash
kubectl -n <namespace> logs -f deploy/connector-service
```

**Order and what to watch**

1. The **connector** runs the database migrations on its first start ([step 7](#7-what-the-database-migrations-do)).
   Let it finish before the other services start. Do not start 2.0 services against the database
   from here on: after the first migration it is no longer a 2.0 database.
2. **live-data, remote-control, mission-autonomy, admin-console** and the **console** start on the
   same 3.0 version — never mix 2.0 and 3.0 core services.
3. The **Integration Hub** starts last. On its first 3.0 start it moves every credential 2.0
   encrypted in its database into OpenBao, then drops the encrypted column. It needs
   `VAULT_ENCRYPTION_KEY` (your 2.0 value) for that start only; without it, and with encrypted
   credentials present, it stops with
   `… credentials are still encrypted in the database: set VAULT_ENCRYPTION_KEY once …` and
   changes nothing. After its log shows `moved N credentials from the database to OpenBao`, remove
   `VAULT_ENCRYPTION_KEY`.

> **Open before release:** the 3.0 Helm chart has no value for `VAULT_ENCRYPTION_KEY` any more. An
> installation whose Integration Hub holds credentials from 2.0 has to give the key to the
> `integration-hub-backend` Deployment by hand for its first start (and take it away again) until
> the chart offers this.

## 7. What the database migrations do

The connector migrates the database from the last 2.0 version to 3.0 in one go. Notable steps:

| Migration | What it does | Reversible |
| --- | --- | --- |
| Audit log, node runs, custom roles, AI assistant, approval rules, simulated runs, agent profiles | new tables and columns | — |
| Webhook tokens | hashes every event trigger's webhook token with the webhook-token pepper and **drops the stored token** | no |
| Claim codes | hashes every asset claim code again with the claim-code pepper | no |
| Single sign-on role mappings | turns each provider's claim-to-role mapping into group mappings and **drops the old column** | no |
| Timestamps | every remaining timestamp column becomes `timestamp with time zone` (values read as UTC). Decompresses the compressed detection chunks first; the compression policy compresses them again later. Takes the longest on a large database | no |
| Keycloak link | adds each user's Keycloak account (`keycloak_subject`) | — |
| Users and single sign-on in Keycloak only | **guard** (step 5), then **drops** `users.password_hash` and the identity provider tables | no |
| Secrets sealed in OpenBao only | **guard** (step 5): refuses while anything is sealed with `EXPORT_PLATFORM_KEK` | — |

Each migration runs in a transaction: one that refuses or fails changes nothing itself, but the
migrations before it stay applied.

On its first start admin-console also converts the stored operator stream keys to the 3.0 form.
Stream keys a 2.0 installation issued **with** `LIVE_STREAM_KEYS_PEPPER` set cannot be converted
and are revoked (logged as `Revoked N stream keys …`): cut new keys for the devices that used them.

## 8. Verify

| Check | How |
| --- | --- |
| Services ready | `docker compose ps` / `kubectl -n <namespace> get pods`: nothing restarting. admin-console: `curl -fsS https://<console-api>/q/health/ready` |
| Migrations | the connector's log reports the migrations applied and no error; it serves requests |
| OpenBao | `bao status`: `Initialized true`, `Sealed false` |
| Keycloak | `curl -s https://<keycloak>/realms/zequent/.well-known/openid-configuration` names the issuer `https://<keycloak>/realms/zequent` |
| Sign-in | open the console: it sends you to Keycloak. Sign in as the system administrator, then as a user of an organization with single sign-on |
| Users in step with Keycloak | as system administrator, `POST /api/admin-console/keycloak/sync`: no conflicts |
| License | **License** / **Organizations** in the console: every organization `VALID`. admin-console logs no `sealed secret(s) could not be moved` |
| Assets and telemetry | existing assets are online and send telemetry; a command reaches a device |
| Pairing | create a claim code and pair a test device ([Pairing after the upgrade](../concepts/upgrading-platform-to-3.0.md#pairing-after-the-upgrade)) |
| Integration Hub | its connections work; the credentials appear under **Manage → Access & Integrations → Secrets** |
| Live video | a stream plays; devices whose stream key was revoked have new keys |
| Backups | take the first OpenBao snapshot and Keycloak database dump ([Backup and restore](secrets-and-sign-in.md#backup-and-restore)) |

## 9. Roll back

The migrations drop columns and tables (stored webhook tokens, password hashes, the identity
provider tables, the Integration Hub's encrypted credentials) and change column types. **The only
way back to 2.0 is restoring the backup**; everything written on 3.0 since is lost.

1. Stop the 3.0 core services, the console and the Integration Hub.
2. Restore Postgres into an empty database with the same TimescaleDB version:

   ```bash
   # Docker Compose; on Kubernetes the same commands in the Postgres pod
   docker compose exec -T postgres psql -U postgres -c 'DROP DATABASE zequent_db' -c 'CREATE DATABASE zequent_db'
   docker compose exec -T postgres psql -U postgres -d zequent_db \
     -c 'CREATE EXTENSION IF NOT EXISTS timescaledb' -c 'SELECT timescaledb_pre_restore()'
   docker compose exec -T postgres pg_restore -U postgres -d zequent_db --no-owner < zequent_db-<date>.dump
   docker compose exec -T postgres psql -U postgres -d zequent_db -c 'SELECT timescaledb_post_restore()'
   ```

3. Restore the Redis snapshot (stop Redis, put the saved `/data` back, start it).
4. Put back the 2.0 configuration — `.env.customer` or values, including `EXPORT_PLATFORM_KEK` and
   `VAULT_ENCRYPTION_KEY` — and the 2.0 image tags, and start 2.0.

OpenBao and Keycloak may keep running; 2.0 does not use them. Edge and client credentials issued on
3.0 are not valid on 2.0.

## 10. Adapters and client applications

Adapters and client applications do not have to move with the platform; see
[What keeps working](../concepts/upgrading-platform-to-3.0.md#what-keeps-working).

> **Open before release:** with OpenBao, the 3.0 platform signs its calls into adapters with the
> Transit key `service-token`. Adapters on a 2.0 Edge SDK verify those calls only against the static
> `ZQNT_PLATFORM_PUBLIC_KEY` and may refuse them. Check your 2.0 adapters against the upgraded
> platform during the rehearsal; adapters on a 3.0 Edge SDK fetch the key set from
> `ZQNT_PLATFORM_KEYS_URL`.

- **Adapters:** move them one at a time to the 3.0 Edge SDK —
  [Java](../edge-sdk/edge-sdk-adapter-3.0.md) · [Python](../edge-sdk/edge-sdk-python-adapter-3.0.md) ·
  [Go](../edge-sdk/edge-sdk-go-adapter-3.0.md). Set `ZQNT_PLATFORM_KEYS_URL` to
  `https://<console-api>/api/admin-console/auth/service-keys`; keep `ZQNT_EDGE_TOKEN` and
  `ZQNT_PLATFORM_PUBLIC_KEY`.
- **Client applications:** after the platform, to the 3.0 Client SDKs —
  [Upgrading to 3.0](../concepts/upgrading-to-3.0.md).
