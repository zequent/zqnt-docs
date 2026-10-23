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
  platform. The exception is creating assets with `registerAsset` — see Breaking changes. Capabilities fall back to the v2 `GetCapabilities` the same way, and 2.0 telemetry
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
| Packages with the generated contracts | `zqnt-utils-java`, `zqnt-utils` (Python), `github.com/zequent/zqnt-utils-golang/v2` (private) — internal platform libraries | the public Zequent API, repository `zequent/zqnt-api`: **`com.zqnt:zqnt-api`** (Java), **`zqnt-api`** (Python, `subdirectory = "python"`), **`github.com/zequent/zqnt-api/go/v3`** (Go) — Apache 2.0, version line `3.0.0` |
| SDK repositories and coordinates | one repository per SDK and language: `com.zqnt.sdk:edge-java-sdk` / `client-java-sdk`, `zqnt-edge-sdk-python` / `zqnt-client-sdk-python`, `…/zqnt-edge-sdk-go/v2` / `…/zqnt-client-sdk-go/v2` (`github.com/Zequent/…`) | one repository per language, Edge and Client SDK side by side: **`com.zqnt:zqnt-sdk-java-edge`** / **`com.zqnt:zqnt-sdk-java-client`** (`zequent/zqnt-sdk-java`), `edge-python-sdk` / `zqnt-client-sdk` from **`zequent/zqnt-sdk-python`** (`subdirectory = "edge"` / `"client"`), **`github.com/zequent/zqnt-sdk-go/v3/edge`** / **`…/v3/client`** (`/v3`, lower-case owner) |
| SDK versions | `2.0.x` | `3.0.0` for every SDK and API package |
| Commands from Client SDKs | one typed method per command | `executeCommand` by command id; the typed 2.x methods are removed from the 3.0 Client SDKs (the 2.0 SDKs keep working against a 3.0 platform) |
| Commands in adapters | one method per command | a command registry; the typed methods are deprecated (adapters that implement them keep working) |
| Human approvals | any credential could decide a `HUMAN_APPROVAL` node | only a **signed-in user** can; service, edge, Integration Hub and Client SDK credentials are refused. The decision records who decided and when |
| Pairing assets | claim codes, plus the legacy paths: the Edge SDKs' `registerAsset` (Java) / `RegisterAsset` (Go) and binding a DJI dock by typing the organization UUID | **claim codes only.** `registerAsset`/`RegisterAsset` are removed from the 3.0 Edge SDKs (use `ensureAsset`/`EnsureAsset` with a claim code), an edge credential can no longer call the 2.x `ConnectorService/RegisterAsset`, and a DJI dock binds only with a claim code from the console. See [Pairing after the upgrade](#pairing-after-the-upgrade) |

### Pairing after the upgrade

- **Assets that already exist stay as they are.** Pairing only matters when an asset is created;
  a device whose asset exists keeps working, whichever way it was paired.
- **A new device** is paired with a one-time claim code created in the console:
  the adapter sets `ZQNT_CLAIM_CODE` and calls `ensureAsset` / `ensure_asset` / `EnsureAsset`, or
  the operator types the code into the DJI dock. The claim decides the organization. Or create the
  asset in the console first; the adapter then binds to it by serial number.
- **A DJI dock that has to be bound again** (reset, new dock) needs a claim code; typing the
  organization UUID is refused. Re-binding a dock whose asset exists keeps it in its organization.
- **A 2.0 adapter that still creates assets with `registerAsset`** gets `PERMISSION_DENIED` from a
  3.0 platform: pair its devices with claim codes, or create their assets in the console, before
  the platform moves to 3.0.
- **The simulator** creates its devices' assets in the organization of its own edge credential:
  give it an edge credential issued for one organization (`ZQNT_EDGE_TOKEN`) and set
  `ORGANIZATION_ID` to that organization. A device requested for another organization is refused.

What the package change means in practice:

- **Java:** replace the `com.zqnt:zqnt-utils-java` dependency (and its repository) with
  `com.zqnt:zqnt-api` from `https://maven.pkg.github.com/Zequent/zqnt-api`, and the SDK artifacts with
  `com.zqnt:zqnt-sdk-java-edge` / `com.zqnt:zqnt-sdk-java-client` from
  `https://maven.pkg.github.com/Zequent/zqnt-sdk-java`. The Java packages (`com.zqnt.utils.*`,
  `com.zqnt.protos.*`, `com.zqnt.sdk.*`) are the same, so no import changes. The SDKs depend on
  `zqnt-api` already; only code that used `zqnt-utils-java` directly needs that change. Do not keep an
  old and a new artifact on one classpath (duplicate classes).
- **Python:** replace `zqnt-utils` with `zqnt-api` (git `https://github.com/zequent/zqnt-api`,
  `subdirectory = "python"`) and the module prefix `zqnt_utils` with `zqnt_api` (for example
  `zqnt_api.generated.zqnt.telemetry.v3`); take the SDKs from `https://github.com/zequent/zqnt-sdk-python`
  with `subdirectory = "edge"` or `"client"`.
- **Go:** change import paths to `github.com/zequent/zqnt-api/go/v3/gen/...` (same package paths
  below `gen/`) and the SDK packages to `github.com/zequent/zqnt-sdk-go/v3/edge/...` and
  `.../v3/client/...`. The modules are public: no `GOPRIVATE` needed for them.

Also new on 3.0: calls that name no organization are counted and logged (metric
`zqnt_tenancy_organization_fallback_total`); setting `zqnt.tenancy.organization-fallback=refuse`
refuses them instead of placing them in the default organization (default: `warn`).

## Secrets and sign-in

3.0 can keep every secret of the platform in OpenBao and sign every person in through Keycloak,
with single sign-on per organization (for example Microsoft Entra ID). Both are optional in the
preview. Password users move into Keycloak with their existing passwords; people who signed in
through their organization's single sign-on on 2.0 keep their accounts. See
[Secrets and sign-in](../admin/secrets-and-sign-in.md).

## What must be upgraded together

- **The core services** — connector, live-data, remote-control, mission-autonomy and admin-console —
  are one release with one version: upgrade them together. A 3.0 command runs remote-control →
  mission-autonomy → adapter, and the v3 telemetry and capability paths span live-data,
  remote-control and connector; mixing 2.x and 3.0 core services is not supported.
- **The console with admin-console**, as with every release (the console is admin-console's client).
- **An SDK with its API package.** A 3.0 SDK needs the `zqnt-api` 3.0 package (Python: pinned to
  the same revision, uv allows one source per package); do not mix it with `zqnt-utils-*` in one
  application.

Adapters and client applications do **not** have to move with the platform — see "What keeps
working".

## Recommended upgrade order

1. **Pair every device with a claim code** while still on 2.0, so nothing depends on the legacy
   pairing paths that 3.0 removes.
2. **Move human approvals to signed-in users.** Any integration that approves `HUMAN_APPROVAL` nodes
   with a client credential has to sign in as a user before the platform moves to 3.0.
3. **Upgrade the platform**: all core services and the console together. 2.0 adapters and 2.0
   client applications keep working against it.
4. **Upgrade adapters**, one at a time, to the 3.0 Edge SDKs (command registry, v3 telemetry with
   catalog keys). The platform picks each one up within 10 minutes, without a restart.
5. **Upgrade client applications** to the 3.0 Client SDKs and move from typed calls to commands by id
   ([Upgrading to 3.0](upgrading-to-3.0.md)).

Step 4 can also come first: 3.0 adapters work against a 2.x platform. 3.0 Client SDKs have no 2.x
command calls: their calls — commands by id, manual control, the telemetry, detection and alert
watches — fail with `UNIMPLEMENTED` until the platform runs 3.0, so client applications move after
the platform.
