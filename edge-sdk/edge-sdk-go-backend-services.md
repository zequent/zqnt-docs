# Edge SDK (Go) — Backend Services

`edgesdk.NewEdgeClient` builds client stubs for the three backend services alongside the
`EdgeAdapter` gRPC server (see [Overview](edge-sdk-go-overview.md)) — access them via
`client.LiveData()`, `client.Connector()`, and `client.MissionAutonomy()`.

## Live Data

`LiveDataService` (`github.com/Zequent/zqnt-edge-sdk-go/livedata`) manages persistent gRPC
client-streaming connections and routes telemetry frames through them — one open stream per device
serial number, reused across calls rather than redialed each time.

| Method | Purpose |
|--------|---------|
| `ProduceTelemetryData(ctx, *domains.TelemetryRequestData)` | Push telemetry using the POJO-style domain type; the SDK maps it to the proto request for you |
| `ProduceTelemetry(ctx, deviceSN, *proto.ProduceTelemetryRequest)` | Push a pre-built proto request directly, for advanced/full-control cases |
| `CloseStream(ctx, deviceSN)` | Close the persistent stream for one device |
| `CloseAllStreams(ctx)` | Close every open stream — call during shutdown (`client.Shutdown` already does this for you) |

```go
client.LiveData().ProduceTelemetryData(ctx, &domains.TelemetryRequestData{
    SN:   "YOUR-DEVICE-SN",
    Type: domains.TelemetryTypeAsset,
    AssetTelemetry: &domains.AssetTelemetryData{
        Latitude:  &lat,
        Longitude: &lon,
        // ... see AssetTelemetryData for the full field set (mirrors the AssetTelemetry proto)
    },
})
```

`domains.TelemetryType` distinguishes dock/primary-asset telemetry (`TelemetryTypeAsset`) from
paired-drone/sub-asset telemetry (`TelemetryTypeSubAsset`).

## Connector

`ConnectorService` (`github.com/Zequent/zqnt-edge-sdk-go/connector`) is this SDK's **old-API**
Connector surface — asset registration and the old Mission/Task/Scheduler CRUD, not the
current-model Connector the Java/Python Edge SDKs and the Go **client** SDK's
[`connector` package](../client-sdk/CONNECTOR_GO.md) expose.

| Category | Methods |
|----------|---------|
| Assets | `GetAssetBySN`, `GetAssetByID`, `GetSubAssetBySN`, `UpdateAsset`, `RegisterAsset`, `DeRegisterAsset` |
| Organizations | `GetOrganizationByID` |

Mission and task lookup are not part of the Go Edge SDK's connector client — unlike the Java and
Python Edge SDKs, it has no `GetTask`. A Go adapter therefore cannot resolve a bare task id, which
is why it can only take the command-based execution path (see
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md)).

```go
asset, err := client.Connector().GetAssetBySN(ctx, "YOUR-DEVICE-SN")
```

## Mission Autonomy

`MissionAutonomyService` (`github.com/Zequent/zqnt-edge-sdk-go/missionautonomy`) is likewise the
**old** Mission/Task model — scheduler and mission/task CRUD driven from the adapter side, not the
Application/Skill execution engine.

| Category | Methods |
|----------|---------|
| Schedulers | `GetScheduler` |

That is the whole surface: the Go Edge SDK's `missionautonomy` package exposes scheduler lookup
only. Creating and managing missions and tasks belongs to the **Client SDK**, used by customer
applications.

