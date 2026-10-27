# Sizing and capacity (3.0, preview)

> **Preview.** Zequent **3.0** is not released yet; this page describes the preview builds and may
> still change.

> **Measured on a production-like setup.** The figures on this page come from load tests against
> Zequent's own 3.0 environment: native images with the CPU and memory limits below, Kubernetes
> (k3s) on one server, Postgres/TimescaleDB, Redis and Keycloak in the same cluster, the load
> generated from outside over the public ingress. The data set is small (a few dozen assets, a few
> thousand runs), so list views on an installation with long history will be slower. Where no
> figure has been measured yet, this page says so.

- [What drives the load](#what-drives-the-load)
- [Service level objectives](#service-level-objectives)
- [What admin-console replicas handle](#what-admin-console-replicas-handle)
- [Long runs and failover](#long-runs-and-failover)
- [When to add replicas](#when-to-add-replicas)
- [CPU and memory per service](#cpu-and-memory-per-service)
- [Postgres and Redis](#postgres-and-redis)
- [Sign-in (Keycloak)](#sign-in-keycloak)
- [Telemetry ingest](#telemetry-ingest)
- [Limits that are configuration, not capacity](#limits-that-are-configuration-not-capacity)
- [Not measured yet](#not-measured-yet)

## What drives the load

| Driver | Unit | Mostly loads |
| --- | --- | --- |
| Connected assets × telemetry rate | samples per second | live-data, Redis, Postgres/TimescaleDB (telemetry persistence) |
| Operators with the console open | open WebSockets, reads per second | admin-console, Redis |
| Application runs | concurrent executions, nodes per second | mission-autonomy, connector, Postgres |
| AI assistant | concurrent chat sessions, tool calls per second | agent service, admin-console, Postgres (audit) |
| AI nodes in runs | concurrent AI nodes per organization | agent service, model provider |
| Stored history | days of telemetry, detections, audit events | Postgres disk and the latency of list views |
| Live video | concurrent streams | media server, network |

Console list views (executions over a time range, audit log, detections) get slower as history
grows, so the same number of operators costs more on an installation with a year of history than on
a new one.

## Service level objectives

These are the targets Zequent measures each 3.0 release against. Latencies are measured at the
client and include network and TLS, from inside or next to the cluster.

| Objective | Target |
| --- | --- |
| Console API reads (assets, executions over a time range, alerts, audit log, detections) | p95 < 300 ms, p99 < 1 s |
| Console API errors | < 1 % of requests |
| Console WebSocket open | ticket p95 < 300 ms, upgrade p95 < 500 ms, > 99 % succeed |
| AI assistant read tools (platform side, without the model) | p95 < 300 ms, p99 < 1 s, > 99 % succeed |
| AI assistant tool calls refused by the rate limit at normal load | none |
| Telemetry ingest | no sample rejected, stream acknowledged with p95 < 500 ms, > 99 % of streams succeed |
| AI assistant tool-call throughput | 2 admin-console replicas ≈ 2 × the calls per second of 1 replica |
| Concurrent chat sessions per admin-console replica | 50 (not measured yet) |

Command round trips to devices, live video, execution throughput and model latency are not covered:
they depend on the device, the network and the model provider.

## What admin-console replicas handle

admin-console serves the console's API and WebSockets, so it is the service that grows with the
number of operators. Load: a mix of console reads (assets, asset relations, executions over a time
range, alerts, audit log, detections, who-am-I) plus open console WebSockets, ramped from 10 to
about 320 reads per second and from 10 to 500 WebSockets over 10 minutes. Every other core service
ran two replicas.

| admin-console replicas (2 CPU, 384 MiB each) | Reads per second | p50 | p95 | p99 | Errors | CPU per replica |
| --- | --- | --- | --- | --- | --- | --- |
| 2 | up to 250 | 22–25 ms | ≤ 83 ms | — | 0 % | up to ~1.6 |
| 2 | 290–320 | 0.6–0.8 s | 2–5 s | — | ~5 % (HTTP 503) | 1.9 (at the limit) |
| 3 | up to 320 | 25 ms | 63 ms | 92 ms | 0 % | ~1.3 |

- Two replicas carry about **250 console reads per second** within the objectives. Beyond that
  their CPU limit is reached and admin-console answers **503 with Retry-After** instead of queueing
  without bound; nothing crashes and the other services stay unaffected.
- A third replica removed every error at the same load: capacity grows with replicas.
- 500 open consoles (WebSockets) next to 320 reads per second: every socket connected (5,559 of
  5,559 upgrades), ticket p95 19 ms.

Plan **one admin-console replica (2 CPU) for about 120 console reads per second**, and keep the
average CPU per replica below about 1 CPU so a load spike has room while a new replica starts.

## Long runs and failover

- **8 hours** at 20 console reads per second with 20 open consoles, one replica of each service:
  p95 143 ms, no failed request, no restart, no latency drift over the night.
- **Failover:** with two replicas of every core service and 50 reads per second, one replica of
  each service was deleted at the same moment: no request failed (0 of 15,950); the replacements
  were ready within a minute.

## When to add replicas

- **admin-console:** add a replica when console reads approach 120 per second per replica, when
  its average CPU stays above about 1 CPU per replica, when console p95 latency stays above 300 ms,
  or when the AI assistant's tool calls approach what the replicas serve. The AI assistant's tool-call rate limit is shared by all replicas, so adding
  replicas does not raise it.
- **Other core services:** add a replica when a service's CPU stays near its limit or the
  latencies in its area (telemetry ack, command dispatch, execution steps) leave the objectives.
  Per-replica figures for these services are not measured yet.
- Before adding replicas, check Postgres: every replica opens its own database connection pool
  (see below).
- **Automatically:** the Helm chart can add and remove core-service replicas on CPU
  (`autoscaling.enabled: true`; the configured replica count is the minimum,
  `autoscaling.maxReplicas` per service the maximum). It needs the Kubernetes metrics API
  (metrics-server). Defaults: a replica is added when the average CPU per replica stays above
  1 CPU for a minute, at most one per minute; one is removed after ten quiet minutes, at most one
  every five minutes.

## CPU and memory per service

Requests and limits of Zequent's own 3.0 environment, with what the services used there:

| Service | CPU request | CPU limit | Memory request | Memory limit | Idle (per replica) | At ~320 reads/s (per replica) |
| --- | --- | --- | --- | --- | --- | --- |
| connector | 50m | 2 | 128 MiB | 384 MiB | ~60m, ~55 MiB | 0.4–1 CPU |
| live-data | 50m | 2 | 96 MiB | 256 MiB | ~50m, ~30 MiB | ~0.1 CPU |
| remote-control | 50m | 2 | 96 MiB | 256 MiB | ~5m, ~50 MiB | ~0.1 CPU |
| mission-autonomy | 100m | 2 | 384 MiB | 768 MiB | ~25m, 190–270 MiB | ~0.1 CPU |
| admin-console | 50m | 2 | 128 MiB | 384 MiB | ~25m, ~35 MiB | ~1.3 CPU (3 replicas) |

At idle a full set of two replicas per core service used about 0.3 CPU and 0.8 GiB memory in total.
Under console load, admin-console and connector carry the work; live-data's share follows the
telemetry rate, mission-autonomy's the number of running Applications. Watch CPU usage and CPU
throttling of the containers when you size, and raise limits before you add replicas to a service
that is throttled.

## Postgres and Redis

Postgres (with TimescaleDB) holds every record and the telemetry history; Redis is the cache, the
pub/sub bus and the live state of every asset.

**Postgres connections.** Count the pool sizes of all services times their replicas, and keep
Postgres' `max_connections` above that sum with headroom for migrations and administration.
admin-console's pool holds up to 50 connections per replica (`DB_POOL_MAX_SIZE`, default `50`).

**Redis connections.** Every core service keeps up to 32 Redis connections per replica
(`REDIS_MAX_POOL_SIZE`, default `32`), with a waiting queue of 128 requests (256 in admin-console,
`REDIS_MAX_POOL_WAITING`).

Measured: at ~320 console reads per second Postgres used 0.4–0.8 CPU and ~1.9 GiB memory, Redis
0.2 CPU and ~130 MiB, with all core services at two or three replicas.

Not measured yet: Postgres sizing for larger data sets, disk per asset-day of telemetry at 1 Hz, the
growth of the audit log (one row per AI assistant tool call), and Redis memory per connected asset
and per open console. Until then, monitor Postgres disk growth during the first weeks of operation
and size retention from it.

## Sign-in (Keycloak)

Every password sign-in costs Keycloak a deliberately expensive hash (Argon2: several megabytes of
memory and real CPU each), so a burst of sign-ins (a shift starting, a load test) loads Keycloak far
more than steady use. The chart bounds Keycloak's work per replica (16 request threads, at most 1,000
queued requests, beyond that 503) and gives it 2 GiB; run **two Keycloak replicas** where many
people sign in at once. Sign-in throughput grows with Keycloak replicas and CPU, not with threads.
Tokens are refreshed without a hash, so signed-in consoles cost little.

## Telemetry ingest

Every adapter streams telemetry to live-data; the platform's target is that no sample is rejected
and each stream is acknowledged within 500 ms (p95). The samples-per-second one live-data replica
handles has not been measured yet. Telemetry load grows with assets × samples per second: 100
assets at 1 Hz is 100 samples per second, the same 100 assets at 10 Hz is 1,000.

## Limits that are configuration, not capacity

| Limit | Setting | Default |
| --- | --- | --- |
| AI assistant tool calls per organization per minute, shared by all admin-console replicas | `ZQNT_AGENT_TOOL_CALLS_PER_MINUTE` | `600` |
| admin-console database connections per replica | `DB_POOL_MAX_SIZE` | `50` |
| Redis connections per service replica | `REDIS_MAX_POOL_SIZE` | `32` |

A refused request because of one of these limits is not a sign that the installation is too small.

## Not measured yet

These figures follow once the production-like measurement is done; this page will be updated with
them:

- reference sizes (small, medium, large installation: assets, operators, concurrent runs, chat
  sessions, replicas, CPU and memory, Postgres and Redis); console reads per replica are measured
  above
- telemetry samples per second per live-data replica
- concurrent executions per mission-autonomy replica
- chat sessions per admin-console replica, and AI nodes per organization
- Postgres and Redis sizing per load, and disk per asset-day of telemetry
