# Edge SDK -- Connector Service

The `ConnectorService` interface gives an edge adapter access to the platform's asset registry over gRPC. It covers what an adapter itself needs — registering and updating its own asset(s), resolving and updating the task it's currently executing, looking up schedulers and organization info, and reporting which commands it supports. **Creating** mission/task records, and managing missions at all, stays a **Client SDK** (customer application) concern — see [Tasks](#tasks) below for the precise, code-verified boundary.

Full method-by-method reference: [Connector API Reference](../api-reference/edge-sdk-connector-reference.md).

## Table of Contents

- [Overview](#overview)
- [Asset Management](#asset-management)
- [Asset Payloads](#asset-payloads)
- [Tasks](#tasks)
- [Schedulers](#schedulers)
- [Organization](#organization)
- [Capabilities](#capabilities)
- [Error Handling](#error-handling)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)

---

## Overview

From the edge adapter, you use `ConnectorService` to:

- Register your asset when the adapter starts, and deregister it on shutdown.
- Update asset state as it changes.
- Resolve a task the platform handed you, and write adapter-computed fields or a status change back onto it.
- Fetch a scheduler's definition and the organization it belongs to.
- Store and retrieve asset payloads (arbitrary versioned metadata blobs, e.g. calibration data).
- Report which commands your adapter currently supports, by implementing `getCapabilities` on `EdgeAdapterService` — this is what lets the Admin Console show only the controls an asset actually implements.

The SDK provides a ready-to-use implementation (`ConnectorServiceImpl`) that handles gRPC communication and Proto-to-DTO mapping.

---

## Asset Management

### Register an Asset

```java
import com.zqnt.utils.asset.domains.AssetDTO;

AssetDTO asset = new AssetDTO();
asset.setSn("YOUR_DEVICE_SN");
asset.setName("Dock Alpha");
asset.setAssetType("ASSET_TYPE_DOCK");
asset.setVendor("DJI");

connectorService.registerAsset(asset)
    .thenAccept(registered -> log.info("Asset registered with ID: {}", registered.getId()))
    .exceptionally(err -> {
        log.error("Failed to register asset", err);
        return null;
    });
```

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
AssetDTO update = new AssetDTO();
update.setSn("YOUR_DEVICE_SN");
update.setName("Dock Alpha -- Updated");

connectorService.updateAsset("550e8400-e29b-41d4-a716-446655440000", update)
    .thenAccept(updated -> log.info("Asset updated"));
```

### Deregister an Asset

```java
connectorService.deRegisterAsset("550e8400-e29b-41d4-a716-446655440000")
    .thenAccept(success -> {
        if (success) log.info("Asset deregistered");
    });
```

---

## Asset Payloads

Store arbitrary metadata alongside an asset or sub-asset — for example, a generated flight-plan artifact or calibration data.

```java
connectorService.upsertAssetPayload("YOUR_DEVICE_SN", null, payloadDTO)
    .thenAccept(saved -> log.info("Payload stored: {}", saved.getId()));
```

---

## Tasks

`EdgeAdapterService`'s `prepareTask`/`startTask` receive only a task ID (see
[Edge Adapter Reference — Task Execution](../api-reference/edge-sdk-adapter-reference.md#task-execution)) — resolve it with
`getTaskById`, then write back onto that same record with `updateTask` as your adapter learns more
(e.g. a generated flight-plan file's URL) or as the task's status changes:

```java
connectorService.getTaskById(taskId)
    .thenCompose(taskDTO -> {
        // ... generate and upload your flight plan from taskDTO.getConfig() ...

        WaypointTaskConfig config = (WaypointTaskConfig) taskDTO.getConfig();
        config.setFileUrl(uploadedFileUrl);
        config.setFileMd5(uploadedFileMd5);

        // Persist it — the platform's response to updateTask does not echo these fields back,
        // so re-fetching afterward would lose them. Keep using this same in-memory taskDTO.
        return connectorService.updateTask(taskDTO.getId().toString(), taskDTO);
    })
    .thenAccept(updated -> log.info("Task prepared: {}", updated.getId()));
```

```java
taskDTO.setStatus(TaskStatus.TASK_RUNNING);
connectorService.updateTask(taskDTO.getId().toString(), taskDTO)
    .thenAccept(updated -> {
        // Now actually trigger the flight on your hardware.
    });
```

**`createTask`/`deleteTask` exist on the interface but have no confirmed real-adapter usage** —
creating and deleting task records is a Client SDK (customer application) responsibility. The same
holds for every Mission method (`getMissionById`/`createMission`/`updateMission`/`deleteMission`) —
mission management stays client-side entirely.

---

## Schedulers

Schedulers define when and how often a task or command runs.

```java
connectorService.getSchedulerById("scheduler-uuid")
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler));

connectorService.createScheduler(schedulerDTO)
    .thenAccept(created -> log.info("Scheduler created: {}", created.getId()));

connectorService.updateScheduler("scheduler-uuid", updatedScheduler)
    .thenAccept(updated -> log.info("Scheduler updated"));

connectorService.deleteScheduler("scheduler-uuid")
    .thenAccept(success -> {
        if (success) log.info("Scheduler deleted");
    });
```

---

## Organization

```java
connectorService.getOrganizationById("org-uuid")
    .thenAccept(org -> log.info("Organization: {}", org.getName()));
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

> **Beta preview — 2.0.x, not yet released.** An unmerged branch adds Skill Registry
> self-reporting to this interface (beyond the live `getCapabilities` snapshot above) and removes
> every Mission/Task method outright. See the
> [2.0.x migration guide](../concepts/migration-guide-2.0.md#per-sdk-impact) for what replaces
> them, or the [2.0.x reference](../api-reference/edge-sdk-connector-reference-2.0.md) for the exact
> methods.

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

```properties
quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010
quarkus.grpc.clients.connector-service.keep-alive-without-calls=true
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.

---

## Usage Examples

### Startup Registration Pattern

```java
import io.quarkus.runtime.StartupEvent;
import io.quarkus.runtime.ShutdownEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@ApplicationScoped
public class AssetRegistration {

    private final ConnectorService connectorService;
    private final EdgeClientConfig config;
    private String registeredAssetId;

    public AssetRegistration(ConnectorService connectorService, EdgeClientConfig config) {
        this.connectorService = connectorService;
        this.config = config;
    }

    void onStart(@Observes StartupEvent event) {
        AssetDTO asset = new AssetDTO();
        asset.setSn(config.sn());
        asset.setAssetType(config.assetType().name());
        asset.setVendor(config.assetVendor().name());

        connectorService.registerAsset(asset)
            .thenAccept(registered -> {
                registeredAssetId = registered.getId();
                log.info("Asset registered: {}", registeredAssetId);
            })
            .exceptionally(err -> {
                log.error("Asset registration failed", err);
                return null;
            });
    }

    void onStop(@Observes ShutdownEvent event) {
        if (registeredAssetId != null) {
            connectorService.deRegisterAsset(registeredAssetId).join();
            log.info("Asset deregistered");
        }
    }
}
```

---

## See also

- [Connector API Reference](../api-reference/edge-sdk-connector-reference.md) — every method, including which ones have confirmed real-adapter usage and which don't
- [Edge Adapter Reference — Task Execution](../api-reference/edge-sdk-adapter-reference.md#task-execution) — how `prepareTask`/`startTask`/`stopTask` reach your adapter in the first place
