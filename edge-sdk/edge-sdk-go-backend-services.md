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
| Missions (old model) | `GetMissionByID`, `CreateMission`, `UpdateMission`, `DeleteMission` |
| Tasks (old model) | `GetTaskByID`, `GetTaskByFlightID`, `CreateTask`, `UpdateTask`, `DeleteTask` |
| Schedulers | `GetSchedulerByID`, `CreateScheduler`, `UpdateScheduler`, `DeleteScheduler` |
| Organizations | `GetOrganizationByID` |

```go
asset, err := client.Connector().GetAssetBySN(ctx, "YOUR-DEVICE-SN")
```

## Mission Autonomy

`MissionAutonomyService` (`github.com/Zequent/zqnt-edge-sdk-go/missionautonomy`) is likewise the
**old** Mission/Task model — scheduler and mission/task CRUD driven from the adapter side, not the
Application/Skill execution engine.

| Category | Methods |
|----------|---------|
| Missions | `CreateMission`, `UpdateMission`, `GetMission`, `DeleteMission` |
| Tasks | `GetTask`, `GetTaskByFlightID`, `CreateTask`, `UpdateTask`, `DeleteTask`, `StartTask`, `StopTask` |
| Schedulers | `GetScheduler`, `CreateScheduler`, `UpdateScheduler`, `DeleteScheduler` |

Multi-step, graph-based automations (Applications/Skills) are authored and triggered through the
**Client SDK**, the same as
for the Java/Python Edge SDKs — this package's old Mission/Task methods predate that model entirely
and aren't a way to reach it.

## Skill Registry (unmerged)

A `skillregistry` package exists on branch `feature/skill-registry-v2` (**not on `main` as of this
writing** — check whether it's landed) that lets an adapter self-report its own command contracts
directly into the platform's persisted Skill Registry, mirroring the Java/Python
`ObserveSkillContract`/`ListSkillContracts` surface:

```go
import (
    "github.com/Zequent/zqnt-edge-sdk-go/skillregistry"
    connectorpb "github.com/Zequent/zqnt-edge-sdk-go/gen/connector/proto"
)

svc := skillregistry.NewServiceImpl(connectorpb.NewConnectorServiceClient(conn), logger)
svc.ObserveSkillContract(ctx, &connectorpb.SkillContractProtoDTO{CommandId: "acme.custom_scan"})
```

**Why this coexists with the old proto tree rather than replacing it**: the SDK's `proto/` submodule
(`zqnt-protos`) is pinned to a schema that predates the Skill/Capability/Application model — no
`ObserveSkillContract`, no `ListSkillContracts`, and `GetCapabilities`'s wire format is still a plain
`available bool` rather than the richer `CapabilityState` enum. Bumping that submodule is a real,
coordinated breaking change to a dependency this repo doesn't own, so `skillregistry` instead vendors
its own up-to-date generated code under `gen/connector/...` (different Go import path, no conflict)
rather than forcing that bump just to add one new package. Practical consequence: until the submodule
itself is bumped, `EdgeAdapter.GetCapabilities` (see [Edge Adapter](edge-sdk-go-adapter.md)) and
`skillregistry.ObserveSkillContract` report device capabilities through **two different, temporarily
coexisting schemas** — don't expect them to line up field-for-field yet.
