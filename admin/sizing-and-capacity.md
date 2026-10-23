# Sizing and capacity (3.0, preview)

> **Preview.** Zequent **3.0** is not released yet; this page describes the preview builds and may
> still change.

> **First measurements only.** The figures on this page come from one load-test run on a single
> laptop-class machine, with every service in development mode and one instance of each. They show
> where one instance tips over; they are not a sizing for a production installation. They will be
> replaced by measurements on a production-like setup (native images, real CPU and memory limits)
> before 3.0 is released. Where no figure has been measured yet, this page says so.

- [What drives the load](#what-drives-the-load)
- [Service level objectives](#service-level-objectives)
- [What one admin-console replica handles](#what-one-admin-console-replica-handles)
- [When to add replicas](#when-to-add-replicas)
- [CPU and memory per service](#cpu-and-memory-per-service)
- [Postgres and Redis](#postgres-and-redis)
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

## What one admin-console replica handles

admin-console serves the console's API and WebSockets, so it is the service that grows with the
number of operators.

Measured setup: one instance of each core service in development mode (JVM, not the native image
you run), on one laptop (Intel i7-12700H, 20 threads, 62 GB RAM) shared with Postgres, Redis,
Keycloak, the load generator and other work. Small data set: 7 assets, 264 executions, 821 audit
events.

| Console reads per second (target) | Served | p50 | p95 | p99 | Errors |
| --- | --- | --- | --- | --- | --- |
| 10 | 10.0/s | 20 ms | 184 ms | 221 ms | 0 % |
| 25 | 24.9/s | 30 ms | 270 ms | 335 ms | 0 % |
| 50 | 50.0/s | 29 ms | 249 ms | 331 ms | 0 % |
| 75 | 63.7/s | 657 ms | 4.3 s | 6.9 s | 3.9 % |
| 100 | 80.4/s | 2.4 s | 5.5 s | 9.5 s | 7.5 % |

- Up to **50 console reads per second** stay within the objectives (p95 ≤ 270 ms, p99 ≤ 335 ms,
  no errors). Between 50 and 75 reads per second the instance saturates: latency grows to seconds
  and some requests fail instead of waiting.
- 50 open consoles (WebSockets) next to 50 reads per second: every socket connected. Read latency
  rose to p95 644 ms because the test reconnected all 50 sockets at the same moment every 30 s,
  which real consoles do not do.
- Starting straight into 50 reads per second caused a 20–30 s stall with errors right after start;
  ramping the load up did not.
- The slowest read is the execution list over a time range (p50 228 ms at 50 reads per second);
  asset status, detections and alerts answer in 13–33 ms at p50.

Plan **one admin-console replica for about 50 console reads per second** until the
production-like measurement replaces this figure.

## When to add replicas

- **admin-console:** add a replica when console reads approach 50 per second per replica, when
  console p95 latency stays above 300 ms, or when the AI assistant's tool calls approach what one
  replica serves. The AI assistant's tool-call rate limit is shared by all replicas, so adding
  replicas does not raise it.
- **Other core services:** add a replica when a service's CPU stays near its limit or the
  latencies in its area (telemetry ack, command dispatch, execution steps) leave the objectives.
  Per-replica figures for these services are not measured yet.
- Before adding replicas, check Postgres: every replica opens its own database connection pool
  (see below).

## CPU and memory per service

No per-service sizing has been measured yet. As a starting point, these are the requests and
limits Zequent's own development environment runs the 3.0 native images with:

| Service | CPU request | CPU limit | Memory request | Memory limit |
| --- | --- | --- | --- | --- |
| connector | 50m | 2 | 128 MiB | 384 MiB |
| live-data | 50m | 2 | 96 MiB | 256 MiB |
| remote-control | 50m | 2 | 96 MiB | 256 MiB |
| mission-autonomy | 100m | 2 | 384 MiB | 768 MiB |
| admin-console | 50m | 2 | 128 MiB | 384 MiB |

These are not validated for production load. Watch CPU usage and CPU throttling of the containers
when you size, and raise requests and limits before you add replicas to a service that is throttled.

## Postgres and Redis

Postgres (with TimescaleDB) holds every record and the telemetry history; Redis is the cache, the
pub/sub bus and the live state of every asset.

**Postgres connections.** Count the pool sizes of all services times their replicas, and keep
Postgres' `max_connections` above that sum with headroom for migrations and administration.
admin-console's pool holds up to 50 connections per replica (`DB_POOL_MAX_SIZE`, default `50`).

**Redis connections.** Every core service keeps up to 32 Redis connections per replica
(`REDIS_MAX_POOL_SIZE`, default `32`), with a waiting queue of 128 requests (256 in admin-console,
`REDIS_MAX_POOL_WAITING`).

Not measured yet: Postgres CPU and memory per load, disk per asset-day of telemetry at 1 Hz, the
growth of the audit log (one row per AI assistant tool call), and Redis memory per connected asset
and per open console. Until then, monitor Postgres disk growth during the first weeks of operation
and size retention from it.

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
  sessions, replicas, CPU and memory, Postgres and Redis)
- telemetry samples per second per live-data replica
- concurrent executions per mission-autonomy replica
- chat sessions per admin-console replica, and AI nodes per organization
- Postgres and Redis sizing per load, and disk per asset-day of telemetry
