# Securing an installation: encrypting the traffic inside it (3.0, preview)

> **Preview.** Zequent **3.0** is not released yet; this page describes the preview builds and may
> still change.

Everything that reaches an installation from outside is encrypted at its edge: the console and the
API behind the ingress or reverse proxy, the client SDKs' gRPC endpoint, Edge Nodes' link. Inside
the installation the services talk to each other over the cluster or Compose network. **Internal
TLS** encrypts that traffic too and authenticates both ends, so that a party inside the network can
neither read nor tamper with telemetry, commands or service tokens.

- [What internal TLS covers](#what-internal-tls-covers)
- [How it works](#how-it-works)
- [PostgreSQL and Redis](#postgresql-and-redis)
- [Switching it on with Helm](#switching-it-on-with-helm)
- [Switching it on with Docker Compose](#switching-it-on-with-docker-compose)
- [Workloads outside core](#workloads-outside-core)
- [The final step: refusing plaintext](#the-final-step-refusing-plaintext)
- [Bringing your own CA](#bringing-your-own-ca)
- [Checking that it works](#checking-that-it-works)
- [What is not covered yet](#what-is-not-covered-yet)

## What internal TLS covers

| Path | Without internal TLS | With internal TLS |
| --- | --- | --- |
| Core service ↔ core service (gRPC) | service token per call, plaintext | **mutual TLS**: each service presents its own certificate, the token is bound to it |
| Ingress / reverse proxy → admin-console API | plaintext inside the cluster | TLS to admin-console's certificate |
| Client SDKs' gRPC ingress → core services | plaintext inside the cluster | TLS to each service's certificate |
| Edge Nodes → edge-link | TLS (edge-link's own certificate) | unchanged; the same listener also serves the core |
| Core services → PostgreSQL | password, plaintext | TLS to PostgreSQL's certificate (`sslmode=verify-full`), then the password |
| Core services → Redis | password, plaintext | TLS to Redis's certificate (`rediss://`), then the password |
| Edge adapters, simulator, Integration Hub → core services (gRPC) | edge credential per call, plaintext | TLS to each service's certificate, then the edge credential |
| Edge adapters, simulator, Integration Hub → Redis / PostgreSQL | password, plaintext | TLS to the store's certificate, then the password |
| Edge workloads → admin-console's key set (HTTP) | plaintext | HTTPS to admin-console's certificate |
| Console (server side) → admin-console API, media server auth hook → admin-console | plaintext inside the cluster | HTTPS to admin-console's certificate |
| Core services → edge adapters (commands) | signed service token per call, plaintext | unchanged (not covered yet) |

Every core service keeps its plaintext port for the health probes and, until the
[final step](#the-final-step-refusing-plaintext), for callers that have not moved to TLS yet.

## How it works

One **internal certificate authority (CA)** per installation issues a certificate to every core
service (connector, live-data, remote-control, mission-autonomy, admin-console, edge-link). The
certificate names the service as its callers reach it (`connector-service`,
`connector-service.<namespace>.svc`, …) and is good for server and client authentication.

A service with `ZQNT_INTERNAL_TLS=true` serves its gRPC API over TLS 1.3 on
`ZQNT_INTERNAL_TLS_PORT` (default `8443`) with that certificate, asks every caller for a client
certificate and verifies a presented one against the CA, and dials the other core services over TLS
on the same port, presenting its own certificate. The files are read from
`/etc/zqnt/internal-tls/{tls.crt,tls.key,ca.crt}` (`ZQNT_INTERNAL_TLS_CERT`, `_KEY`, `_CA`) and
re-read every `ZQNT_INTERNAL_TLS_RELOAD_PERIOD` (default `10m`): a renewed certificate is picked
up by new connections without a restart. Private keys are PKCS#8 PEM.

A **platform service token** (what one core service sends another) is bound to the caller's
certificate once the final step is on: captured on the network or from a log, the token is
worthless without the key of the pod it was issued to. A person's token and an edge credential are
not bound to a certificate; their calls need TLS, not a client certificate.

## PostgreSQL and Redis

The same CA issues a server certificate to the installation's PostgreSQL and Redis, named as the
services reach them (`postgres-v2`, `redis-v2`; `postgres`, `redis` on Compose). With
`ZQNT_INTERNAL_TLS=true` a core service connects to PostgreSQL with `sslmode=verify-full` on both
of its pools (the reactive one and Flyway's) and to Redis on its TLS port with `rediss://`, each
time checking the server's certificate against the CA **and** the server's name, so that nothing
inside the network can pose as the database or the cache. The password still authenticates the
service afterwards; neither store asks the services for a client certificate.

Redis keeps serving its plaintext port next to the TLS one, and PostgreSQL keeps accepting
plaintext sessions, until the [final step](#the-final-step-refusing-plaintext), for a workload
that has not moved yet (an edge adapter of your own on an older SDK). A core service whose Redis
URL still says `redis://` while internal TLS is on connects with `rediss://` rather than silently
falling back to plaintext.

For a managed PostgreSQL or Redis outside the installation, which the internal CA cannot vouch
for, set `ZQNT_INTERNAL_TLS_POSTGRES=false` or `ZQNT_INTERNAL_TLS_REDIS=false` (Helm:
`internalTls.postgres.enabled`, `internalTls.redis.enabled`): the service then keeps its own
connection settings for that store, which you configure for its certificate as before.

## Switching it on with Helm

The chart needs cert-manager (it already issues the ingress certificates). In your values:

```yaml
internalTls:
  enabled: true
```

The upgrade creates the installation's CA (`Issuer zqnt-internal-ca`, Secret `zqnt-internal-ca`),
a `Certificate` per core workload (Secret `<workload>-internal-tls`, 90 days, renewed 30 days
before the end, key rotated on every renewal), mounts them, sets `ZQNT_INTERNAL_TLS=true`, adds the
TLS port `grpcs` to the Services and NetworkPolicies, and moves the gRPC ingress and the console API
ingress to the services' TLS listeners. The chart's own PostgreSQL (`postgres-v2`) and Redis
(`redis-v2`) get a certificate each and serve TLS: PostgreSQL with `ssl=on` on its usual port,
Redis on `internalTls.redis.port` (6380) next to its plaintext port; the core services connect
that way, and the stores restart once for it. Nothing else changes: the plaintext ports stay for
the probes and for callers outside core.

The workloads outside core — the edge adapters (DJI, AI, SAPIENT, ONVIF), the simulator, the
Integration Hub, the console and the media server — get the CA certificate alone, the `ca.crt`
item of the CA Secret (`internalTls.ca`), mounted at `/etc/zqnt/internal-tls/ca.crt`, and are
pointed at the TLS listeners: see [Workloads outside core](#workloads-outside-core). Their images
must carry the change (the 3.1 line of the adapters, simulator, Integration Hub and console);
switch `internalTls.enabled` on together with them, not before.

A renewed store certificate is read at the next restart of `postgres-v2` / `redis-v2`
(`kubectl rollout restart deployment/postgres-v2 -n <namespace>`); the core services reload theirs
on their own.

With edge-link enabled, `services.edgeLink.tls.port` must equal `internalTls.port` (both `8443` by
default): edge-link has one TLS listener for nodes and the core, with both certificates on it and
SNI choosing per client.

## Switching it on with Docker Compose

Issue the CA and the certificates once, then add the overlay to every `docker compose` call:

```bash
scripts/internal-tls/make-certs.sh internal-tls
docker compose --env-file .env.customer \
  -f docker-compose.customer.yml -f docker-compose.internal-tls.yml up -d
```

`make-certs.sh` writes `internal-tls/ca.crt` and `internal-tls/<service>/{tls.crt,tls.key,ca.crt}`
for every core service, for `postgres` and for `redis` with openssl (OpenBao's PKI engine works
just as well: issue a certificate per service with `DNS:<service>` and write the same three files).
The overlay mounts each service's directory, sets `ZQNT_INTERNAL_TLS=true`, starts PostgreSQL with
`ssl=on` and Redis with a TLS port 6380 next to its plaintext port, points the core services at
`rediss://redis:6380`, and verifies Keycloak's database connection against the CA as well. The
containers read the files as their own user, so keep the directory where only the installation's
operator can reach it. The certificates are good for 90 days: run the script again before that and
restart the services; the CA is kept.

The admin-console API stays published in plaintext on 8005 for the reverse proxy in front of it;
8443 serves the same API over TLS with the internal CA. The overlay also moves the simulator
(`--profile simulator`) to the core services' TLS ports, Redis's TLS port and admin-console's key
set over HTTPS, and the console's server-side proxy to `https://admin-console-service:8443`, each
with `internal-tls/ca.crt` mounted alone: the plaintext addresses in `.env.customer` stay as they
are and are overridden by the overlay.

## Workloads outside core

Everything that calls the core from inside the installation without being a core service — the
edge adapters, the simulator, the Integration Hub, the console's server side and the media
server's auth hook — reads the same two settings:

| Variable | Meaning |
| --- | --- |
| `ZQNT_INTERNAL_TLS=true` | call the core services, Redis, PostgreSQL and admin-console's key set over TLS, each connection verified against the CA **and** the dialled host name; off (the default) is plaintext |
| `ZQNT_INTERNAL_TLS_CA` | the CA certificate, default `/etc/zqnt/internal-tls/ca.crt`; a missing file stops the workload at start instead of falling back to plaintext |

They present **no client certificate**: their edge credential (`ZQNT_EDGE_TOKEN`), the console
session or the hook secret authenticates them as before, now over TLS. The Helm chart and the
Compose overlay set both variables and the TLS addresses (`…:8443`, `rediss://…:6380`,
`sslmode=verify-full` on the Integration Hub's database URL, the key set on
`https://…:8443/api/admin-console/auth/service-keys`); an adapter you run elsewhere, built on
the Java, Python or Go Edge SDK (3.1 or later), takes the same two variables plus the CA file,
and the client SDKs read them too for a client inside the installation.

The Edge SDKs keep plaintext as the default, so an adapter on an older SDK keeps working until the
[final step](#the-final-step-refusing-plaintext); from then on it is refused.

## The final step: refusing plaintext

`ZQNT_INTERNAL_TLS_REQUIRE=true` (Helm: `internalTls.require: true`; Compose: in `.env.customer`)
closes the door: a gRPC call that arrives over plaintext is refused before its credential is read,
and a platform service token is accepted only together with the caller's client certificate. On
Helm the same switch closes Redis's plaintext port and makes PostgreSQL reject a session without
TLS; on Compose set `ZQNT_REDIS_PLAINTEXT_PORT=0` and `ZQNT_POSTGRES_PLAINTEXT=reject` in
`.env.customer` alongside it.

The order that keeps the installation working at every step:

1. **Roll out the images** of every workload with its internal TLS change — the core services
   (3.0.1 or later), then the edge adapters, the simulator, the Integration Hub and the console
   (their 3.1 images) — while internal TLS is still off; nothing changes yet.
2. **Switch internal TLS on** (`internalTls.enabled: true`; Compose: issue the certificates and
   add the overlay). The core services serve TLS next to their plaintext ports and call each other
   over mutual TLS; the workloads outside core move to the TLS listeners; the plaintext ports
   stay open. Check the log lines and `pg_stat_ssl` / `CLIENT LIST` below: nothing of yours should
   still appear on a plaintext port, apart from the health probes, the metrics scrape and the agent
   service (see [What is not covered yet](#what-is-not-covered-yet)).
3. **Move what is still on plaintext**: an edge adapter of your own gets the two variables and
   the CA file; a client of your own inside the installation the same, or stays on the public
   gRPC ingress, which is TLS already.
4. **Refuse plaintext**: `internalTls.require: true` (Compose: the three `.env.customer`
   settings). From now on a plaintext gRPC call is answered with `UNAUTHENTICATED`, Redis's
   plaintext port is closed and PostgreSQL rejects a plaintext session; the health probes keep
   their plaintext HTTP port, which serves no API.

Until step 4 an installation with internal TLS on already encrypts every call between the core
services, the ingresses, the console API, the database and the cache, and from the edge
workloads, and only tolerates callers that have not moved yet. To go back one step, switch
`require` off again; the certificates stay.

## Bringing your own CA

Helm: set `internalTls.issuer.create: false` and name your cert-manager issuer in
`internalTls.issuer` (`kind` and `name`), for example a `Vault` issuer backed by OpenBao's PKI
engine or an issuer for a corporate CA. The Certificates then come from it; what the services need
is `tls.crt`, `tls.key` (PKCS#8) and `ca.crt` in each workload's Secret, which cert-manager writes
for CA-type issuers.

Compose: write the three files per service yourself; the paths can be changed with
`ZQNT_INTERNAL_TLS_CERT`, `ZQNT_INTERNAL_TLS_KEY` and `ZQNT_INTERNAL_TLS_CA`.

## Checking that it works

A service logs `Internal TLS on: dialling [...] over mutual TLS on port 8443` at start and
`Listening on: http://0.0.0.0:<port> and https://0.0.0.0:8443`. From a pod or container with
openssl:

```bash
openssl s_client -connect connector-service:8443 -CAfile /etc/zqnt/internal-tls/ca.crt </dev/null
```

shows the service's certificate, `Verify return code: 0 (ok)` and `TLSv1.3`. With `require` on, a
plaintext gRPC call is answered with `UNAUTHENTICATED: This installation accepts gRPC calls over TLS
only`, and a platform token without a client certificate with `UNAUTHENTICATED: A platform service
token needs the caller's client certificate`. On Kubernetes, `kubectl get certificate -n <namespace>`
lists every workload certificate with `READY True`.

For the stores: in PostgreSQL, `select usename, ssl, version from pg_stat_ssl join
pg_stat_activity using (pid) where client_addr is not null` lists every session with `ssl = t` and
`TLSv1.3` for the core services and the Integration Hub (a plaintext one shows `f`);
`redis-cli -p 6380 --tls --cacert ca.crt ping` answers `PONG`, and `CLIENT LIST` on the TLS port
shows the services', adapters' and simulator's connections while the plaintext port only has those
of the workloads that have not moved yet.

For a workload outside core: it starts without a certificate error and its telemetry keeps
arriving; an adapter with `ZQNT_INTERNAL_TLS=true` but no CA file refuses to start and names
`ZQNT_INTERNAL_TLS_CA`. From its pod, `openssl s_client -connect connector-service:8443 -CAfile
/etc/zqnt/internal-tls/ca.crt </dev/null` shows `Verify return code: 0 (ok)`.

## What is not covered yet

- **The agent service.** It still calls admin-console over plaintext gRPC and is the one
  workload the final step would cut off; it moves in its own change.
- **Commands to the adapters.** The platform dials an adapter's own gRPC port (its
  `EdgeAdapterService`) over plaintext; every command carries a signed, short-lived service token
  the adapter verifies, but the connection is not encrypted yet. The Integration Hub's bridge port
  is the same case.
- **Keycloak's own database on Kubernetes.** The chart's Keycloak keeps its separate PostgreSQL
  (`keycloak-postgres`) in plaintext; on Compose, where Keycloak shares the platform's PostgreSQL,
  its connection is already verified against the CA.
- **Metrics.** The metrics endpoint stays plaintext for Prometheus, on the management port that
  serves nothing else.
