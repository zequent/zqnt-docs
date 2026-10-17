# Secrets and sign-in (3.0, preview)

> **Preview.** Zequent **3.0** is not released yet; this page describes the preview builds and may
> still change. On 2.0, see [Organizations, Users & Single Sign-On](organizations-and-sso.md).

In 3.0 every secret of the platform lives in **OpenBao** (an open-source secrets manager, API
compatible with HashiCorp Vault), and every person signs in through **Keycloak**. Both are optional
while 3.0 is in preview: an installation without them keeps its secrets in the environment of the
services and signs people in through admin-console, as on 2.0.

- [How secrets are kept](#how-secrets-are-kept)
- [How people sign in](#how-people-sign-in)
- [Development keys are refused](#development-keys-are-refused)
- [Rotating keys and secrets](#rotating-keys-and-secrets)
- [Backup and restore](#backup-and-restore)
- [Microsoft Entra ID for an organization](#microsoft-entra-id-for-an-organization)

## How secrets are kept

### Switching a service to OpenBao

`ZQNT_SECRETS_PROVIDER=openbao` switches a core service (connector, live-data, remote-control,
mission-autonomy, admin-console) from its environment to OpenBao. The default is `env`.

| Variable | Meaning |
| --- | --- |
| `ZQNT_SECRETS_PROVIDER` | `env` (default) or `openbao` |
| `OPENBAO_ADDR` | OpenBao's address, e.g. `http://openbao:8200` |
| `OPENBAO_KUBERNETES_ROLE`, `OPENBAO_KUBERNETES_TOKEN_FILE` | Kubernetes: the role the service logs in as, and its ServiceAccount token |
| `OPENBAO_APPROLE_ROLE_ID` / `OPENBAO_APPROLE_SECRET_ID` (or `…_FILE` with a path) | Docker Compose: the service's AppRole login |
| `ZQNT_SECRETS_OPENBAO_STARTUP_TIMEOUT` | How long a service waits for OpenBao at start (default `PT2M`) |

Each workload has its own login and its own policy, and reads only its own paths. No private
signing key is kept in any service: tokens and credentials are signed inside OpenBao.

### Secrets (KV, mount `zqnt/`)

| Path | Fields | Read by |
| --- | --- | --- |
| `infrastructure/postgres` | `password` | connector, admin-console, Integration Hub |
| `infrastructure/redis` | `password` | every core service, the Integration Hub, the DJI adapter |
| `infrastructure/object-storage` | `platform-access-key`, `platform-secret-key`, `dock-access-key`, `dock-secret-key`, `adapter-access-key`, `adapter-secret-key` | connector, the DJI adapter |
| `infrastructure/mqtt` | `username`, `password`, `dock-username`, `dock-password` | the DJI adapter |
| `infrastructure/mediamtx` | `hook-secret` | admin-console |
| `infrastructure/terrain` | `bearer-token` | mission-autonomy |
| `infrastructure/simulator-control` | `token` | admin-console |
| `infrastructure/grafana` | `admin-password` | admin-console |
| `platform/peppers/webhook-tokens`, `platform/peppers/claim-codes`, `platform/peppers/stream-keys` | `value` | the core services |
| `platform/admin-console/keycloak` | `client-secret` | admin-console (client `zqnt-admin-console`) |
| `platform/edge-adapter-dji/cloud-api` | `app-key`, `app-id`, `app-license` | the DJI adapter |
| `organizations/<organization id>/integration/<id>` | the connection's credentials | the Integration Hub |

The peppers are the server-side keys under which webhook tokens, asset claim codes and operator
stream keys are stored as HMAC-SHA-256 hashes. No stored token, code or key can be read back.

### Signing and encryption keys (Transit, mount `zqnt-transit/`)

| Key | Type | Used for |
| --- | --- | --- |
| `service-token` | ed25519 | Short-lived tokens the core services call each other with |
| `credential` | ed25519 | Edge and client credentials issued by admin-console |
| `export-signing` | ed25519 | `.zqnt` exports; each export names the key version it was signed with |
| `platform-kek` | aes256-gcm96, derived per organization | Encrypting stored license activations and AI model keys |

The public halves are published at `GET /api/admin-console/auth/service-keys` (a JWK set, one entry
per key version, identified by `kid`). Adapters, the simulator and the Integration Hub verify the
platform's tokens against it: set `ZQNT_PLATFORM_KEYS_URL` to that address. They fetch the set
every 5 minutes, and again (at most every 30 seconds) when a token names a key they do not know.

### What the console shows

**Manage → Access & Integrations → Secrets** lists the organization's secrets — credentials,
stream keys, claim codes, webhook tokens, AI model keys, the license activation, single sign-on and
the Integration Hub's connections — with their kind, owner, creation, expiry and last use. It
never shows a stored value; a value is shown once, when it is created. Reading needs
`secrets:read`, revoking and rotating `secrets:manage`. A system admin also sees the **Platform
keys** tab: each signing and encryption key with its current version, when it was last rotated
and which versions are in use.

### Docker Compose

`docker-compose.customer.yml` starts OpenBao with `--profile openbao`:

- `openbao-seal-key` generates a 32-byte seal key once into the volume `openbao_seal`; OpenBao
  unseals itself with it on every start.
- `openbao-init` initialises OpenBao on the first start (recovery key and root token stay in
  `init.json` in the volume `openbao_init`, nothing secret is logged) and, on every start, creates
  the mounts, the Transit keys, one policy and one AppRole per workload (role-id and secret-id in
  the volume `openbao_credentials`), and writes the KV secrets above **once**, from
  `.env.customer`, for each path that does not exist yet. Peppers you did not set are generated.

The platform service images pinned in the compose file still read their secrets from
`.env.customer`.

### Kubernetes (Zequent Helm chart)

With `openbao.enabled: true` the chart runs OpenBao as a Raft StatefulSet (`openbao.replicas`: 1,
or 3 or 5 for high availability) that initialises itself, without a root token. Each workload logs
in with Kubernetes auth as the role of its ServiceAccount. A configuration Job runs after every
install and upgrade: it creates the mounts, keys, policies and roles, and writes the KV secrets
**once** from the Secrets the chart used before (`<release>-credentials` and the `existingSecret`s
you named). Afterwards OpenBao is the source of every value: the core services and the DJI adapter
no longer get the infrastructure passwords, the peppers, the Keycloak client secret or the
service and export signing keys in their environment.

## How people sign in

Keycloak runs one realm, `zequent`. Each platform organization is a Keycloak **Organization**
whose alias is the platform organization's id. The realm has two clients:

| Client | Kind | Purpose |
| --- | --- | --- |
| `zqnt-console` | public, authorization code with PKCE (S256) | The console signs people in. Redirects to the console's URL |
| `zqnt-admin-console` | confidential, service account | admin-console keeps the realm's organizations, users and identity providers in step. Its secret is the only Keycloak secret the platform holds |

With Keycloak sign-in on (`ZQNT_AUTH_KEYCLOAK_ENABLED=true` on every core service,
`ZQNT_AUTH_KEYCLOAK_ISSUER=https://<keycloak>/realms/zequent`):

- Every core service accepts the realm's access tokens directly: it checks the signature (keys from
  the realm's JWKS by `kid`), the issuer, the audience `zqnt-platform` and that it is an access token.
  The person is the platform user linked to the token's subject.
- **Roles, permissions and site scope stay in the platform.** Nothing in the token grants a role.
- admin-console keeps Keycloak in step: organizations, memberships, and local password users,
  which move **with their existing password hashes** (nobody has to set a new password). It does so
  immediately on every change in the console, completely every 15 minutes
  (`ZQNT_AUTH_KEYCLOAK_SYNC_INTERVAL`), and on `POST /api/admin-console/keycloak/sync` (system
  admin). A Keycloak account the platform did not create is never taken over; it is reported as a
  conflict.
- Disabling a user, changing their roles or revoking their sessions also ends their Keycloak
  sessions.
- admin-console's own login, refresh and password endpoints answer `410 Gone`.

The realm's defaults: passwords of at least 12 characters, not the username or email, not one of
the last three; brute-force protection; access tokens valid for 5 minutes; sessions end after
30 minutes idle and after 10 hours at most.

### Keycloak in Docker Compose

`--profile keycloak` starts Keycloak on port `8180` with its own database `keycloak` on the
platform's Postgres, and imports the realm on the first start. Set in `.env.customer`:

| Variable | Meaning |
| --- | --- |
| `KEYCLOAK_URL` | Keycloak's public URL (your TLS reverse proxy in front of port 8180) |
| `ZQNT_CONSOLE_URL` | The console's public URL, where `zqnt-console` redirects back to |
| `KEYCLOAK_DB_PASSWORD`, `KEYCLOAK_ADMIN_PASSWORD`, `KEYCLOAK_ADMIN_CONSOLE_CLIENT_SECRET` | Generate each once with `openssl rand -hex 32` |

`KEYCLOAK_ADMIN_PASSWORD` is the password of a temporary administrator `admin` for the first
start: create a permanent administrator in Keycloak and delete the temporary one. With
`--profile openbao`, `openbao-init` also writes `KEYCLOAK_ADMIN_CONSOLE_CLIENT_SECRET` to
`platform/admin-console/keycloak` once.

## Development keys are refused

The platform's source and its published images contain keys and secrets for local development.
A service started outside development mode refuses to start when any of these is configured: the
signing keys, the export keys, the platform KEK, the license public key, the stream-hook secret, the
peppers, the Keycloak client secret of `zqnt-admin-console`, or the OpenBao login. With OpenBao the
peppers and the Keycloak client secret are checked as read from OpenBao. The error names the
setting, never the value. Generate your own values for every one of them.

## Rotating keys and secrets

| What | How | Effect |
| --- | --- | --- |
| Transit signing keys `service-token`, `credential` | `bao write -f zqnt-transit/keys/<name>/rotate` | The services sign with the new version once it is 15 minutes old (`ZQNT_SECRETS_OPENBAO_NEW_KEY_VERSION_GRACE`), so every service and adapter has learned it first (the services refresh the keys every 5 minutes, `ZQNT_SECRETS_OPENBAO_KEY_REFRESH`). Older versions keep verifying |
| Transit `export-signing` | `bao write -f zqnt-transit/keys/export-signing/rotate` | New exports name the version they were signed with; older exports keep verifying while their version exists |
| Transit `platform-kek` | `bao write -f zqnt-transit/keys/platform-kek/rotate` | admin-console moves every stored secret to the newest version every hour (`ZQNT_SECRETS_OPENBAO_REWRAP_EVERY`) |
| A secret in KV | change it where it is used (for example the database role's password), then `bao kv patch -mount=zqnt <path> <field>=…` and restart the workloads that read it | — |
| `platform/admin-console/keycloak` | Keycloak: **Clients → zqnt-admin-console → Credentials → Regenerate**, then `bao kv patch -mount=zqnt platform/admin-console/keycloak client-secret=…` | admin-console reads it whenever it requests a new admin token; no restart |
| Keycloak's realm signing keys | Keycloak: **Realm settings → Keys**: add a new key, keep the old one passive until the last tokens signed with it have expired (5 minutes), then remove it | The services fetch the realm's keys every 10 minutes, and again when a token names an unknown key |
| An organization's Entra ID client secret | [see below](#rotating-the-client-secret) | — |
| The peppers | **never** | Changing a pepper invalidates every webhook token, claim code or stream key stored under it |

Edge and client credentials stay valid as long as the key version that signed them exists. Do not
remove old versions of `credential` (`min_decryption_version`, `min_available_version`) before
every credential signed with them has been issued again. For `platform-kek`, raise
`min_decryption_version` only after an hourly run logged nothing left behind (no
`sealed secret(s) could not be moved` warning).

Keep values off the command line (and out of the shell history): write a field as
`<field>=@<file>`, or `<field>=-` to read it from standard input.

### The seal key

The seal key decrypts all of OpenBao's storage. Keep it separately from every backup (for
example in your operators' password manager): a snapshot cannot be restored without it, and a
lost seal key loses every secret. Never let it be generated again for an existing installation.
In production use a KMS or HSM seal (`seal "awskms"`, `"gcpckms"`, `"azurekeyvault"`, `"pkcs11"` or
`"transit"`) instead of the static key; see "Seal migration" in the OpenBao documentation. To
rotate the static key, configure the new key as `current_key` and the old one as `previous_key`
with its `previous_key_id` ("static seal" in the OpenBao documentation).

## Backup and restore

### OpenBao

A Raft snapshot holds every secret, policy and key, encrypted with the seal key; it can only be
restored with the same seal key. Take one daily, store it off the host, and test a restore into an
empty installation with the same seal key at least once a quarter.

Docker Compose (`BAO_TOKEN` = the root token from `init.json` in the volume `openbao_init`, read
into the variable without printing it):

```bash
export BAO_TOKEN=$(docker run --rm -v <project>_openbao_init:/i:ro alpine \
  sed -n 's/.*"root_token": *"\([^"]*\)".*/\1/p' /i/init.json)

# Backup
docker compose exec -T -e BAO_TOKEN openbao sh -c \
  'bao operator raft snapshot save /tmp/openbao.snap && cat /tmp/openbao.snap && rm /tmp/openbao.snap' \
  > openbao-$(date +%F).snap

# Restore (replaces the current state)
docker compose exec -T -e BAO_TOKEN openbao sh -c \
  'cat > /tmp/openbao.snap && bao operator raft snapshot restore -force /tmp/openbao.snap; rm -f /tmp/openbao.snap' \
  < openbao-2026-10-17.snap
```

Back up the volumes `openbao_seal` and `openbao_init` once, separately from the snapshots and off
this host.

Kubernetes: the chart's OpenBao has no root token; an administrator logs in as the ServiceAccount
`openbao-operator`, then runs the same `bao operator raft snapshot` commands in `openbao-0`:

```bash
K="kubectl -n <namespace>"
export BAO_TOKEN=$($K exec -i openbao-0 -- bao write -field=token auth/kubernetes/login role=operator jwt=- \
  <<< "$($K create token openbao-operator --duration 10m)")
$K exec openbao-0 -- env BAO_TOKEN="$BAO_TOKEN" sh -c \
  'bao operator raft snapshot save /tmp/openbao.snap && cat /tmp/openbao.snap && rm /tmp/openbao.snap' \
  > openbao-$(date +%F).snap
```

Tokens issued after a snapshot are invalid after restoring it; the services log in again.

### Keycloak

Keycloak's database holds the users, password hashes, organizations, identity providers and
sessions: back it up like the platform database.

```bash
# Docker Compose: database "keycloak" on the platform's Postgres
docker compose exec -T postgres pg_dump -U postgres -Fc keycloak > keycloak-$(date +%F).dump

# Kubernetes: its own keycloak-postgres
$K exec deploy/keycloak-postgres -- pg_dump -U keycloak -Fc keycloak > keycloak-$(date +%F).dump
```

To restore, stop Keycloak, run `pg_restore --clean -d keycloak` with the dump, and start it again.

A realm export (configuration and users as JSON, for example to compare or to move) runs inside
the running Keycloak with its own management port and a local cache:

```bash
/opt/keycloak/bin/kc.sh export --realm zequent --file /tmp/zequent.json --http-management-port=9001 --cache=local
```

The export contains password hashes and client secrets: treat it as a secret.

## Microsoft Entra ID for an organization

With Keycloak sign-in, each organization can sign its people in through its own Microsoft Entra ID
tenant (or any OpenID Connect provider). Keycloak brokers the sign-in: people type their email on
the sign-in page and, when its domain belongs to the organization, are sent to their Entra ID.

Organization admins configure their own organization (permission `identity:edit`); a system admin
can configure any organization. In the console, open **Single Sign-On** at
`https://<your console>/manage/single-sign-on`.

### 1. Register an app in Entra ID

In the Microsoft Entra admin center, **App registrations → New registration**:

1. **Supported account types:** *Accounts in this organizational directory only* (single tenant).
   Shared tenants (`common`, `organizations`, `consumers`) are refused.
2. **Redirect URI:** platform **Web**, the **Redirect URI** the console's Single Sign-On screen
   shows. It has the form
   `https://<keycloak>/realms/zequent/broker/sso-<organization id>/endpoint`.
3. From **Overview**, note the **Application (client) ID** and the **Directory (tenant) ID**.
4. **Certificates & secrets → Client secrets → New client secret.** Copy the secret's **Value**
   (shown once). Note its expiry date.

Every user needs an email address in Entra ID: a sign-in without one is refused. Keycloak requests
the scopes `openid profile email`.

### 2. Give people their roles: use app roles

The platform turns a claim of the ID token into roles. Use the **`roles`** claim with app roles:

1. In the app registration, **App roles → Create app role**: *Allowed member types* Users/Groups,
   and a **Value** per platform role you want to hand out, for example `Zequent.OrgAdmin`,
   `Zequent.Operator`, `Zequent.Viewer`.
2. In **Enterprise applications → <your app> → Users and groups**, assign users or groups to those
   app roles. Entra ID puts the values of the assigned app roles into the `roles` claim.

Do not use the `groups` claim for this unless you have to. Above 200 groups Entra ID no longer
sends the groups in the token (it sends an overage marker instead), and those people get no role
from their groups. The `groups` claim also carries group object IDs, not names. If you use it
anyway, enable it under **Token configuration → Add groups claim** (security groups) and map the
object IDs.

### 3. Configure the organization in the console

On **Single Sign-On**:

| Field | Value |
| --- | --- |
| Identity provider | **Microsoft Entra ID** |
| Tenant ID | The **Directory (tenant) ID**, or the tenant's primary domain (`contoso.onmicrosoft.com`) |
| Client ID | The **Application (client) ID** |
| Client Secret | The secret's **Value**. Afterwards the field shows *Configured. Leave blank to keep it* |
| Email Domains | The domains of your people's email addresses, comma-separated. A domain can belong to one organization only |
| Group Claim | `roles` |
| Enabled | on |

The client secret is stored in Keycloak only; neither the platform nor the console ever shows it
again. The **Secrets** screen lists it as configured.

Then, under **Group → Role** on the same screen, map each app role **Value** (exactly as written in
Entra ID) to a platform role — a built-in or a custom role, for the whole organization or for
selected sites.

### What happens at sign-in

- At every sign-in Keycloak copies the claim's values to the account, and the platform applies the
  organization's mappings: roles a mapping gives are added, roles it no longer gives are removed.
  Roles an administrator assigned by hand stay.
- A person whose values match no mapping gets the role `viewer`.
- The organization's last organization admin keeps that role even when Entra ID no longer gives
  it; the refusal is recorded in the audit log.
- A person signing in for the first time gets a platform account and takes a license seat. When no
  seat is free, or the account is disabled, the sign-in is refused and the sign-in page shows why.
- People who already signed in through the organization's single sign-on on 2.0 are linked to
  their existing platform account, never duplicated.

### Rotating the client secret

Entra ID client secrets expire. Before the current one does, create a new client secret in the app
registration, enter its value in **Client Secret** on **Single Sign-On** and save, then delete the
old secret in Entra ID.

### Other OpenID Connect providers

Choose **OpenID Connect** and enter the provider's **Issuer URL**; Keycloak reads its discovery
document at `<issuer>/.well-known/openid-configuration`. Register the same redirect URI at the
provider, and set **Group Claim** to the ID token claim that carries the groups or roles (default
`groups`).
