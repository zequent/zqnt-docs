# Edge SDK (Go) — Backend Services

`edgesdk.NewEdgeClient` builds client stubs for the three backend services alongside the
`EdgeAdapter` gRPC server (see [Overview](edge-sdk-go-overview.md)) — access them via
`client.LiveData()`, `client.Connector()`, and `client.MissionAutonomy()`. Every call carries the
adapter's edge credential (`ZQNT_EDGE_TOKEN`).

## Live Data

`LiveDataService` (`github.com/Zequent/zqnt-edge-sdk-go/v2/livedata`) manages persistent gRPC
client-streaming connections and routes telemetry frames through them — one open stream per device
serial number, reused across calls rather than redialed each time.

| Method | Purpose |
|--------|---------|
| `ProduceTelemetryData(ctx, *domains.TelemetryRequestData)` | Push telemetry using the POJO-style domain type; the SDK maps it to the proto request for you |
| `ProduceTelemetry(ctx, deviceSN, *proto.ProduceTelemetryRequest)` | Push a pre-built proto request directly, for advanced/full-control cases |
| `CloseStream(ctx, deviceSN)` | Close the persistent stream for one device |
| `CloseAllStreams(ctx)` | Close every open stream — call during shutdown (`client.Shutdown` already does this for you) |
| `PublishCommandExecutionEvent(ctx, *domains.CommandExecutionEvent)` | Report a command's progress and outcome — see [Edge Adapter — Reporting a command's outcome](edge-sdk-go-adapter.md#reporting-a-commands-outcome) |
| `CloseNotificationStream(ctx)` | Close the notification stream — call during shutdown |

```go
lat, lon := float32(47.3769), float32(8.5417)
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

`ConnectorService` (`github.com/Zequent/zqnt-edge-sdk-go/v2/connector`) covers asset lookup and
registration, and organization lookup — narrower than the Java and Python Edge SDKs' Connector: no
pairing-code methods, no Skill Registry and no schedulers.

| Category | Methods |
|----------|---------|
| Assets | `GetAssetBySN`, `GetAssetByID`, `GetSubAssetBySN`, `UpdateAsset`, `RegisterAsset`, `DeRegisterAsset` |
| Organizations | `GetOrganizationByID` |

`DeRegisterAsset(ctx, sn)` deletes the asset with that serial number — don't call it on shutdown.

```go
asset, err := client.Connector().GetAssetBySN(ctx, "YOUR-DEVICE-SN")
```

## Mission Autonomy

`MissionAutonomyService` (`github.com/Zequent/zqnt-edge-sdk-go/v2/missionautonomy`) is a single read:

| Category | Methods |
|----------|---------|
| Schedulers | `GetScheduler` |

`GetScheduler(ctx, schedulerID, sn)` reads a scheduler definition; a scheduler now targets a Skill or a
single command directly. Work reaches your adapter as commands — see [Edge Adapter](edge-sdk-go-adapter.md).
Authoring and running Applications and Skills belongs to the Admin Console and the **Client SDK**.

