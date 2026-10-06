# Edge SDK -- Connector Service

The `ConnectorService` interface gives an edge adapter access to the platform's asset registry over gRPC. It covers what an adapter itself needs — pairing and updating its own asset(s), reading schedules, reporting the commands it supports to the Skill Registry, and registering media files it uploaded. The 1.3 Mission and Task methods are gone: work reaches an adapter as commands (see [Edge Adapter — Custom Commands](edge-sdk-adapter.md#custom-commands)).

Full method-by-method reference: [Connector API Reference](../api-reference/edge-sdk-connector-reference.md).

## Table of Contents

- [Overview](#overview)
- [Asset Management](#asset-management)
- [Asset Payloads](#asset-payloads)
- [Skill Registry](#skill-registry)
- [Media Files](#media-files)
- [Schedulers](#schedulers)
- [Capabilities](#capabilities)
- [Error Handling](#error-handling)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)

---

## Overview

From the edge adapter, you use `ConnectorService` to:

- Make sure your asset exists when the adapter starts — pairing it with a one-time code if the platform doesn't know it yet.
- Update asset state as it changes.
- Read a schedule's definition.
- Report the commands your adapter supports to the Skill Registry.
- Register a media file the device uploaded.
- Store and retrieve asset payloads (arbitrary versioned metadata blobs, e.g. calibration data).
- Report which commands your adapter currently supports, by implementing `getCapabilities` on `EdgeAdapterService` — this is what lets the Admin Console show only the controls an asset actually implements.

The SDK provides a ready-to-use implementation (`ConnectorServiceImpl`) that handles gRPC communication and Proto-to-DTO mapping; your adapter creates it (see [Quickstart — Wire the SDK](edge-sdk-quickstart.md#step-3b-wire-the-sdk)).

---

## Asset Management

### Pair the Asset at Startup

An adapter does not create assets. Its asset is either created in the Admin Console, or **paired**
with a one-time pairing code (Admin Console, Assets page, **Pairing codes**). `ensureAsset` covers both:
it looks the serial number up, and redeems the code only if the asset is unknown — a code is single-use,
so after the first pairing there is nothing left to redeem.

```java
connectorService.ensureAsset(asset, pairingCode)   // pairingCode may be null
    .thenAccept(found -> log.info("Asset: {}", found == null ? "not on the platform yet" : found.getId()));
```

- `redeemAssetClaim(code, asset)` trades a code for an asset directly. The organization the asset lands
  in comes from the code, never from `asset`. It completes with `null` for every refusal alike —
  unknown, expired, revoked, exhausted, or not valid for this kind of device.
- `describeAssetClaim(code)` returns the name of the organization a code would pair into, without
  spending it — so a device can ask its operator to confirm first.
- `registerAsset` is **deprecated**: it creates an asset with no organization, which no tenant can see.

See [Usage Examples](#usage-examples) for a complete startup class.

### Get Asset by Serial Number / ID

```java
connectorService.getAssetBySn("YOUR_DEVICE_SN")
    .thenAccept(asset -> log.info("Found asset: {} (ID: {})", asset.getName(), asset.getId()));

connectorService.getAssetById("550e8400-e29b-41d4-a716-446655440000")
    .thenAccept(asset -> log.info("Asset SN: {}", asset.getSn()));
```

### Get Sub-Asset by Serial Number

Retrieve a sub-asset (e.g. the drone paired to a dock):

```java
connectorService.getSubAssetBySn("YOUR_DEVICE_SNXXX")
    .thenAccept(subAsset -> log.info("Sub-asset model: {}", subAsset.getModel()));
```

### Update an Asset

```java
AssetDTO update = AssetDTO.builder()
    .sn("YOUR_DEVICE_SN")
    .name("Dock Alpha -- Updated")
    .build();

connectorService.updateAsset("550e8400-e29b-41d4-a716-446655440000", update)
    .thenAccept(updated -> log.info("Asset updated"));
```

### Deregister an Asset

`deRegisterAsset(id)` deletes the asset record. Do not call it on shutdown: a paired asset would have
to be paired again with a new code.

---

## Asset Payloads

Report the hardware mounted on an asset or sub-asset — a camera, gimbal or sensor, with its slot, model and firmware.

```java
connectorService.upsertAssetPayload("YOUR_DEVICE_SN", null, payloadDTO)
    .thenAccept(saved -> log.info("Payload stored: {}", saved.getId()));
```

---

## Schedulers

A schedule defines when a Skill or a single command runs, and on which asset. Schedules are created
and changed in the Admin Console; an adapter can read one:

```java
connectorService.getSchedulerById("scheduler-uuid")
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler));
```

---

## Capabilities

An adapter reports which commands it supports by implementing `getCapabilities(String sn)` on
`EdgeAdapterService`, returning a `CurrentCapabilities` of `Capability` entries. The platform calls
this when it needs to know what an asset can do — for example so the Admin Console can hide
controls an asset does not support, rather than failing at execution time.

```java
@Override
public CompletableFuture<CurrentCapabilities> getCapabilities(String sn) {
    Capability takeoff = new Capability("takeOff", "Take off to a target point",
            true, null, Map.of("source", "adapter"));
    takeoff.setTargetType(CapabilityTargetType.CAPABILITY_TARGET_TYPE_ASSET);
    takeoff.setInputSchema(Map.of("type", "object"));

    return CompletableFuture.completedFuture(
            CurrentCapabilities.of(sn, AssetTypeEnum.ASSET_TYPE_AIRCRAFT, Set.of(takeoff)));
}
```

Return `CurrentCapabilities.empty(sn)` for an asset you do not recognise. See
[Edge Adapter](edge-sdk-adapter.md) for the full `EdgeAdapterService` surface.

---

## Skill Registry

Beyond the live `getCapabilities` snapshot, an adapter can report its commands to the platform's
persisted Skill Registry — one entry per `(command_id, schema_version)`, which Skill authors build on:

| Method | Purpose |
| --- | --- |
| `observeSkillContract(contract)` | Add a contract, or refresh one already known |
| `listSkillContracts(status, commandId)` | List the registry; with `commandId`, that command's version history |
| `setSkillContractStatus(id, status)` | Move a contract through `ACTIVE` / `DRAFT` / `DEPRECATED` / `RETIRED` |
| `setSkillContractPermissions(id, requiredPermissions)` | Replace its required permissions (stored, not yet enforced) |

See the [Connector reference](../api-reference/edge-sdk-connector-reference.md#skill-registry).

---

## Media Files

`registerMediaFile(request)` records a file the device uploaded to the platform's inbox bucket. The
platform attributes it to the execution that was running on the asset and files it under its
organization. Reporting the same object twice is harmless.

---

## Error Handling

All `ConnectorServiceImpl` methods follow a consistent error handling pattern:

1. The gRPC response includes a `hasErrors` flag.
2. If `hasErrors` is `true`, the method logs the error and returns `null` (for entity methods) or `false` (for delete methods).
3. If the gRPC call itself fails (network error, timeout), the `CompletableFuture` completes exceptionally.

```java
connectorService.getAssetBySn("SOME_SN")
    .thenAccept(asset -> {
        if (asset == null) {
            log.warn("Asset not found or server error");
            return;
        }
        // use asset
    })
    .exceptionally(err -> {
        log.error("gRPC call failed", err);
        return null;
    });
```

---

## Configuration

The Connector address is `grpc.client.connector.host` / `.port` (`CONNECTOR_SERVICE_HOST` /
`CONNECTOR_SERVICE_PORT`, default port `8010`). See the [Configuration Guide](edge-sdk-configuration.md#grpc-client-configuration).

---

## Usage Examples

### Startup Pairing Pattern

```java
package com.example.edge;

import com.zqnt.sdk.edge.connector.application.ConnectorService;
import com.zqnt.utils.asset.domains.AssetDTO;
import com.zqnt.utils.common.proto.AssetTypeEnum;
import com.zqnt.utils.common.proto.AssetVendor;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import lombok.extern.slf4j.Slf4j;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.Optional;

@Slf4j
@ApplicationScoped
public class AssetPairing {

    private final ConnectorService connectorService;

    @ConfigProperty(name = "zequent.edge.sn")
    String sn;

    // One-time pairing code from the Admin Console; only needed for a device the platform doesn't know yet
    @ConfigProperty(name = "zqnt.claim-code")
    Optional<String> claimCode;

    public AssetPairing(ConnectorService connectorService) {
        this.connectorService = connectorService;
    }

    void onStart(@Observes StartupEvent event) {
        AssetDTO asset = AssetDTO.builder()
                .sn(sn)
                .name("Dock Alpha")
                .type(AssetTypeEnum.ASSET_TYPE_DOCK)
                .vendor(AssetVendor.ASSET_VENDOR_DJI)
                .build();

        connectorService.ensureAsset(asset, claimCode.orElse(null))
                .thenAccept(found -> {
                    if (found == null) {
                        log.warn("Asset {} is not on the platform yet: create it in the console or set a pairing code", sn);
                    } else {
                        log.info("Asset {} is {}", sn, found.getId());
                    }
                })
                .exceptionally(err -> {
                    log.error("Asset lookup failed", err);
                    return null;
                });
    }
}
```

`zqnt.claim-code` reads `ZQNT_CLAIM_CODE`. Once the asset is paired, the code can be removed.

---

## See also

- [Connector API Reference](../api-reference/edge-sdk-connector-reference.md) — every method
- [Edge Adapter — Custom Commands](edge-sdk-adapter.md#custom-commands) — how work reaches your adapter in 2.0
