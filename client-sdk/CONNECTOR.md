# Zequent Client SDK — Connector

`client.connector()` gives your application direct access to the platform's system of record: asset lookups, organization info, scheduler management, technical configuration, operational policies, and asset payloads. Most integrations only need a handful of these — asset lookup and scheduler management are the most common.

Full method-by-method reference: [Connector API Reference](../api-reference/client-sdk-connector.md).

For Python, see [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md).

## Requests share a context

Every Connector request carries an optional `ConnectorRequestContext` for transaction correlation — you don't need to fill it in for normal use; the SDK generates a transaction ID automatically if you leave it unset.

```java
import com.zqnt.sdk.client.connector.domains.*;

var request = GetAssetBySnRequest.builder()
    .context(ConnectorRequestContext.builder().sn("YOUR_DEVICE_SN").build())
    .build();

client.connector().getAssetBySn(request)
    .thenAccept(response -> System.out.println("Asset: " + response));
```

## Looking up an asset

```java
var request = GetAssetByIdRequest.builder()
    .assetId("550e8400-e29b-41d4-a716-446655440000")
    .build();

client.connector().getAssetById(request)
    .thenAccept(response -> System.out.println(response));
```

Asset payload storage (arbitrary versioned metadata — flight-plan artifacts, calibration data),
organization lookup, and scheduler CRUD follow the same request/response shape — see the
[reference](../api-reference/client-sdk-connector.md) for the full method list.

## Missions and tasks are records, not flights

> These methods create and read mission/task **records**. Creating one does not fly anything by
> itself. How a flight is actually triggered depends on the adapter — see
> [Waypoint Missions](WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use).

> **For real use, create missions and waypoint tasks through `client.missionAutonomy()` instead** —
> confirmed against the backend, its `createMission`/`createTask` route-optimize (and, for a task,
> expand around the mission's no-fly zones) before writing the exact same record shown below. The
> example here still works — same record, no optimization — but `missionAutonomy()` is what you
> want day to day. See the [Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy.md).

```java
import com.zqnt.utils.missionautonomy.domains.MissionDTO;
import com.zqnt.utils.mission.proto.MissionType;

var mission = MissionDTO.builder()
    .name("North Perimeter Patrol")
    .type(MissionType.MISSION_TYPE_PERIMETER_PATROL)
    .build();

var request = CreateMissionRequest.builder().mission(mission).build();
client.connector().createMission(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            log.warn("Create mission failed: {}", response.getError().getErrorMessage());
            return;
        }
        System.out.println("Mission created: " + response.getMissionId());
    });
```

### No-fly zones

A mission's no-fly zones (NFZ) tell the platform's route planner where it must route drones — and dock-return paths — around. A zone's `area` can be a polygon, bounding box, circle, or raw GeoJSON, controlled by `GeoAreaDTO.type` (full requirements per type in the [reference](../api-reference/client-sdk-connector.md#geoareadtotype-requirements)):

```java
import com.zqnt.utils.missionautonomy.domains.*;
import com.zqnt.utils.mission.proto.GeoAreaType;
import com.zqnt.utils.mission.proto.MissionZoneType;
import com.zqnt.utils.mission.proto.ZoneEnforcementType;

var zone = MissionZoneDTO.builder()
    .name("Substation exclusion")
    .type(MissionZoneType.MISSION_ZONE_TYPE_NO_FLY)
    .enforcementType(ZoneEnforcementType.ZONE_ENFORCEMENT_TYPE_HARD_BLOCK)
    .area(GeoAreaDTO.builder()
        .type(GeoAreaType.GEO_AREA_TYPE_CIRCLE)
        .center(GeoPointDTO.builder().latitude(52.520008).longitude(13.404954).build())
        .radiusMeters(150.0)
        .build())
    .active(true)
    .build();

var request = UploadMissionZonesRequest.builder()
    .missionId(missionId)
    .zones(List.of(zone))
    .replaceExisting(false)
    .build();

client.connector().uploadMissionNfzZones(request)
    .thenAccept(response -> System.out.println("Zones uploaded: " + response.isSuccess()));
```

### Correlating a task with an external flight record

```java
var request = GetTaskByFlightIdRequest.builder()
    .flightId("FLIGHT-20260901-0001")
    .build();

client.connector().getTaskByFlightId(request)
    .thenAccept(response -> System.out.println("Task: " + response.getTaskId()));
```

`getTaskByFlightId` looks up a task by the external flight ID an edge adapter assigned it (`TaskDTO.externalTaskId`) — useful for correlating a vendor-side flight record back to its Zequent task without knowing the platform's task ID up front.

> **Beta preview — 2.0.x, not yet released.** Every Mission/Task method above becomes a
> `@Deprecated` stub that fails immediately on an unmerged branch. See the
> [2.0.x migration guide](../concepts/migration-guide-2.0.md#per-sdk-impact) for what replaces
> them.

## Checking what an asset supports

```java
client.remoteControl().getCapabilities(sn)
    .thenAccept(snapshot -> snapshot.getCapabilities()
            .forEach(c -> System.out.println(c.getCommandId() + " -> " + c.getState())));
```

Each entry carries the command id, a description, its target type, and an input schema. Use it to
build a UI that only shows commands the asset actually implements. What an asset reports comes from
its edge adapter — see [Edge SDK — Connector](../edge-sdk/edge-sdk-connector.md#capabilities).

## Error handling

Connector responses carry a `hasErrors` flag rather than throwing for expected business errors (not found, validation failure). Handle both:

```java
client.connector().getAssetBySn(request)
    .thenAccept(response -> {
        if (response.getHasErrors()) {
            log.warn("Asset lookup failed: {}", response.getError().getErrorMessage());
            return;
        }
        // use response
    })
    .exceptionally(err -> {
        log.error("Connector call failed", err);
        return null;
    });
```

## See also

- [Connector API Reference](../api-reference/client-sdk-connector.md) — every method, grouped by area
- [Waypoint Missions](WAYPOINT_MISSIONS.md) — which adapter uses which execution path
- [Functional Responses](FUNCTIONAL_RESPONSES.md) — what a response actually confirms
