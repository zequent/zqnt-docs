# Upgrading the platform to 3.0 (preview)

> **Preview.** Zequent **3.0** is not released yet; **2.0.x** stays the current long-term-support
> line and its documentation is unchanged. This page is for operators and customers planning the
> move of an installation — platform, adapters and integrations — from 2.0 to 3.0. It describes the
> preview builds; details may still change before the release.

For what changes inside a **Client SDK** integration, see [Upgrading to 3.0](upgrading-to-3.0.md).
For adapters, see the 3.0 Edge SDK guides: [Java](../edge-sdk/edge-sdk-adapter-3.0.md) ·
[Python](../edge-sdk/edge-sdk-python-adapter-3.0.md) · [Go](../edge-sdk/edge-sdk-go-adapter-3.0.md).

## What 3.0 is

3.0 adds a second set of contracts, **v3**, next to the 2.x ones (v2). In v3 every action on an
asset is a capability with a dotted command id (`flight.takeoff`, `dock.open_cover`,
`vendor.<vendor>.<name>`) and JSON Schemas, run through one `ExecuteCommand` — there is no RPC per
command any more. Telemetry becomes one sample shape (`TelemetrySample`) with shared fields plus
device-specific `details` under declared keys.

A 3.0 platform serves **both**:

| Service | v3 (new) | v2 (still served) |
| --- | --- | --- |
| remote-control | `zqnt.control.v3.RemoteControlService` for clients; `zqnt.edge.v3.EdgeGatewayService` for adapters (capability reports, command events) | the 2.x `RemoteControlService` |
| live-data | `zqnt.telemetry.v3.TelemetryService` (watch) and `TelemetryIngestService` (adapters publish) | the 2.x `LiveDataService` (ingest and streams) |
| towards adapters | v3 `EdgeAdapterService.ExecuteCommand` | v2 `EdgeAdapterService` |

## What keeps working

- **2.0 adapters against a 3.0 platform.** The platform calls an adapter's v3 `ExecuteCommand`
  first. A 2.0 adapter answers `UNIMPLEMENTED` — nothing ran — and the command goes out again as v2
  `SendCustomCommand`, which every 2.0 Edge SDK routes to the adapter's handler for that id. The
  platform remembers such an asset for 10 minutes, so a 2.0 adapter costs one extra round trip per
  10 minutes, not per command; an adapter upgraded to 3.0 is picked up without restarting the
  platform. Capabilities fall back to the v2 `GetCapabilities` the same way, and 2.0 telemetry
  keeps arriving over the v2 ingest.
- **3.0 adapters against a 2.x platform.** The 3.0 Edge SDKs serve v2 next to v3 from the same
  registrations, and every call they make into the platform (capability reports, command events,
  telemetry) falls back to the v2 call on `UNIMPLEMENTED` — for 10 minutes, then they try v3 again.
  Device-specific telemetry `details` arrive on a 2.x platform only as far as v2 has fields for
  them (see the Edge SDK guides).
- **2.0 client applications against a 3.0 platform.** The 2.x client contracts stay served; 2.0
  Client SDKs keep working (with the one exception under Breaking changes: approvals). The 3.0
  command service runs on remote-control next to the 2.x one, on the same host and port.
- **Configuration and credentials.** Client credentials (`ZQNT_CLIENT_TOKEN`), edge credentials
  (`ZQNT_EDGE_TOKEN`), the platform's public key on adapters and the service endpoints stay as they
  are.

## Breaking changes

| What | 2.0 | 3.0 |
| --- | --- | --- |
| Packages with the generated contracts | `zqnt-utils-java`, `zqnt-utils` (Python), `github.com/zequent/zqnt-utils-golang/v2` (private) — internal platform libraries | **`com.zqnt:zqnt-api-java`**, **`zqnt-api`** (`zqnt-api-python`), **`github.com/zequent/zqnt-api-go/v3`** — public, Apache 2.0, version line `3.0.0` |
| Go module paths | `…/zqnt-edge-sdk-go/v2`, `…/zqnt-client-sdk-go/v2` (`github.com/Zequent/…`) | **`github.com/zequent/zqnt-edge-sdk-go/v3`**, **`github.com/zequent/zqnt-client-sdk-go/v3`** (`/v3`, lower-case owner) |
| SDK versions | `2.0.x` | `3.0.0` for every SDK and API package |
| Commands from Client SDKs | one typed method per command | `executeCommand` by command id; the typed 2.x methods are deprecated (still there, and they still work against 2.x platforms) |
| Commands in adapters | one method per command | a command registry; the typed methods are deprecated (adapters that implement them keep working) |
| Human approvals | any credential could decide a `HUMAN_APPROVAL` node | only a **signed-in user** can; service, edge, Integration Hub and Client SDK credentials are refused. The decision records who decided and when |
| Pairing assets | claim codes, plus the legacy paths: the Java Edge SDK's `registerAsset` and binding by organization UUID | **planned:** claim codes only — `registerAsset` and the organization-UUID bind are to be removed (not done in the preview yet). Move devices to [claim-code pairing](../edge-sdk/edge-sdk-connector.md) now |

What the package change means in practice:

- **Java:** replace the `com.zqnt:zqnt-utils-java` dependency (and its repository) with
  `com.zqnt:zqnt-api-java`. The Java packages (`com.zqnt.utils.*`, `com.zqnt.protos.*`) are the same,
  so no import changes. The SDKs depend on it already; only code that used `zqnt-utils-java`
  directly needs the change.
- **Python:** replace `zqnt-utils` with `zqnt-api` and the module prefix `zqnt_utils` with
  `zqnt_api` (for example `zqnt_api.generated.zqnt.telemetry.v3`).
- **Go:** change import paths to `github.com/zequent/zqnt-api-go/v3/gen/...` (same package paths
  below `gen/`) and the SDK modules to `/v3`. The modules are public: no `GOPRIVATE` needed for them.

Also new on 3.0: calls that name no organization are counted and logged (metric
`zqnt_tenancy_organization_fallback_total`); setting `zqnt.tenancy.organization-fallback=refuse`
refuses them instead of placing them in the default organization (default: `warn`).

## What must be upgraded together

- **The core services** — connector, live-data, remote-control, mission-autonomy and admin-console —
  are one release with one version: upgrade them together. A 3.0 command runs remote-control →
  mission-autonomy → adapter, and the v3 telemetry and capability paths span live-data,
  remote-control and connector; mixing 2.x and 3.0 core services is not supported.
- **The console with admin-console**, as with every release (the console is admin-console's client).
- **An SDK with its API package.** A 3.0 SDK needs the `zqnt-api-*` 3.0 package (Python: pinned to
  the same ref, uv allows one source per package); do not mix it with `zqnt-utils-*` in one
  application.

Adapters and client applications do **not** have to move with the platform — see "What keeps
working".

## Recommended upgrade order

1. **Pair every device with a claim code** while still on 2.0, so nothing depends on the legacy
   pairing paths that 3.0 plans to remove.
2. **Move human approvals to signed-in users.** Any integration that approves `HUMAN_APPROVAL` nodes
   with a client credential has to sign in as a user before the platform moves to 3.0.
3. **Upgrade the platform**: all core services and the console together. 2.0 adapters and 2.0
   client applications keep working against it.
4. **Upgrade adapters**, one at a time, to the 3.0 Edge SDKs (command registry, v3 telemetry with
   catalog keys). The platform picks each one up within 10 minutes, without a restart.
5. **Upgrade client applications** to the 3.0 Client SDKs and move from typed calls to commands by id
   ([Upgrading to 3.0](upgrading-to-3.0.md)).

Step 4 can also come first: 3.0 adapters work against a 2.x platform. 3.0 Client SDKs keep their
2.x calls (deprecated), which work against a 2.x platform, but do **not** switch to them on their
own: their v3 calls — commands by id, manual control, the telemetry watch — fail with
`UNIMPLEMENTED` until the platform runs 3.0.
