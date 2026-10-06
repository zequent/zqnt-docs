# Zequent Client SDK — Connector

`client.connector()` gives your application access to the platform's system of record: your organization's assets, the payloads mounted on them, and the Skill Registry. A client credential (`ZQNT_CLIENT_TOKEN`) reaches only its own organization's assets. Schedules, policies, technical configuration and organizations are administration: they are managed in the Admin Console, and a client credential is refused (`PERMISSION_DENIED`) — see [what a client credential may call](../api-reference/client-sdk-connector.md#what-a-client-credential-may-call).

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

Updating an asset, and listing or updating the payloads mounted on it (camera, gimbal, sensor),
follow the same request/response shape — see the
[reference](../api-reference/client-sdk-connector.md) for the full method list.

## Skill Registry

The Skill Registry is the platform's persisted catalog of every command an edge adapter has
reported, one entry per `(commandId, schemaVersion)`, each with a lifecycle status. Read it to see
which commands exist across your fleet — not only on the assets that are online right now:

```java
client.connector().listSkillContracts(null, null)   // (status, commandId) — either may be null
    .thenAccept(contracts -> contracts.forEach(c ->
            System.out.println(c.getCommandId() + " v" + c.getSchemaVersion() + " " + c.getStatus())));
```

Changing the registry (`observeSkillContract`, `setSkillContractStatus`,
`setSkillContractPermissions`) is refused for a client credential — the registry is platform-wide.

Missions and tasks are gone in 2.0: automated work is built as Applications and Skills — see
[Applications & Skills](../concepts/applications-and-skills.md). No-fly zones are managed in the
Admin Console and apply to every flight the platform commands — see
[No-fly zones and safe returns](../concepts/airspace-safety.md).

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

Connector responses carry a `success` flag rather than throwing for expected business errors (not found, validation failure). Handle both:

```java
client.connector().getAssetBySn(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
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
- [Applications & Skills](../concepts/applications-and-skills.md) — building and running automated work
- [Waypoint Missions](WAYPOINT_MISSIONS.md) — flying a waypoint route
- [Functional Responses](FUNCTIONAL_RESPONSES.md) — what a response actually confirms
