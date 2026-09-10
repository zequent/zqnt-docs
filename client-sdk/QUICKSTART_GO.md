# Zequent Client SDK - Quick Start Guide (Go)

## For Customers: Using the SDK in Your Go Project

This guide shows you how to use the Zequent Client SDK in a Go application. See
[QUICKSTART.md](QUICKSTART.md) / [QUICKSTART_PYTHON.md](QUICKSTART_PYTHON.md) for the Java/Python
equivalents — the underlying gRPC contract is identical, only the calling convention differs.

## Step 1: Add the module

```bash
go env -w GONOSUMDB="github.com/Zequent/*"
go env -w GONOPROXY="github.com/Zequent/*"
go get github.com/Zequent/zqnt-client-sdk-go@latest
```

Requires Go 1.26+ (this module's own `go.mod` pins `go 1.26.2`).

## Step 2: Dial each service you need

Unlike the Java SDK's single auto-configured `ZequentClient`, the Go SDK is **one package per
backend service** (`connector`, `missionautonomy`, `remotecontrol`, `livedata`), each wrapping a
plain `grpc.ClientConnInterface` you dial yourself — there's no CDI/DI container to inject into, and
no built-in retry/circuit-breaker/Stork layer (see "What this SDK does *not* do" below). Dial once
per backend service and keep the connection alive for the life of your process.

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    "github.com/Zequent/zqnt-client-sdk-go/remotecontrol"
)

conn, err := grpc.NewClient("localhost:8002", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

rc := remotecontrol.New(conn)
```

For TLS in production, pass real transport credentials instead of `insecure.NewCredentials()`.

## Step 3: Call it

Every method takes a `context.Context` first and returns `(*Response, error)` — no builder pattern,
no `CompletableFuture`/async wrapping (call it from a goroutine yourself if you need that). The SDK
converts the wire-level `has_errors`/`error` convention into an idiomatic Go `error` for you, so a
non-nil `err` covers both transport failures and platform-side business errors — you don't need to
separately check a `HasErrors` flag the way the Java/Python SDKs do.

```go
package main

import (
    "context"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"

    "github.com/Zequent/zqnt-client-sdk-go/remotecontrol"
    devicecontrol "github.com/Zequent/zqnt-client-sdk-go/gen/devicecontrol/contracts/proto"
)

func main() {
    conn, err := grpc.NewClient("localhost:8002", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    rc := remotecontrol.New(conn)
    ctx := context.Background()

    coordinate := &devicecontrol.GeoCoordinate{Latitude: 47.3769, Longitude: 8.5417, Altitude: 100}
    if _, err := rc.TakeOff(ctx, "YOUR_DEVICE_SN", coordinate); err != nil {
        log.Fatalf("takeoff failed: %v", err)
    }

    if _, err := rc.ReturnToHome(ctx, "YOUR_DEVICE_SN", 0 /* use the device's default RTH altitude */); err != nil {
        log.Fatalf("return-to-home failed: %v", err)
    }
}
```

## Available packages

### `remotecontrol` — manual flight/dock control gateway

```go
rc := remotecontrol.New(conn)   // dial remote-control-service, default port 8002

rc.TakeOff(ctx, sn, coordinate)
rc.GoTo(ctx, sn, coordinate)
rc.ReturnToHome(ctx, sn, altitude)
rc.EnterManualControl(ctx, sn, clientID, userID, sessionID)
rc.LookAt(ctx, ...)
rc.OpenCover(ctx, sn) / rc.CloseCover(ctx, sn)
rc.StartCharging(ctx, sn) / rc.StopCharging(ctx, sn)
rc.RebootAsset(ctx, sn)
rc.GetAssetRuntime(ctx, sn, assetID)
rc.GetCapabilities(ctx, sn)
```

Shares its request/response shapes with `EdgeAdapterService` (`devicecontrol` package) — the same
types flow end to end from this client through to the edge adapter that actually talks to the
hardware.

### `missionautonomy` — missions, tasks & schedulers

```go
ma := missionautonomy.New(conn)   // dial mission-autonomy-service, default port 8004

ma.CreateMission(ctx, mission) / ma.UpdateMission(ctx, id, mission)
ma.GetMission(ctx, missionID)   / ma.DeleteMission(ctx, missionID)

ma.CreateTask(ctx, task) / ma.UpdateTask(ctx, id, task) / ma.DeleteTask(ctx, taskID)
ma.GetTask(ctx, taskID)  / ma.GetTaskByFlightID(ctx, flightID)

ma.StartTask(ctx, taskID) / ma.StopTask(ctx, taskID)
ma.PauseTask(ctx, taskID) / ma.ResumeTask(ctx, taskID)

ma.ListSchedulers(ctx)
```

See [Waypoint Missions](WAYPOINT_MISSIONS.md) for which of these actually flies an asset — the task
lifecycle reaches the device only on adapters that implement it.

### `connector` — assets, schedulers, policies, config

```go
c := connector.New(conn)   // dial connector-service, default port 8010

c.GetAssetBySn(ctx, sn)

// Schedulers
c.GetScheduler(ctx, schedulerID) / c.ListSchedulers(ctx)
c.CreateScheduler(ctx, scheduler) / c.CreateSchedulers(ctx, schedulers)
c.UpdateScheduler(ctx, schedulerID, scheduler)
c.DeleteScheduler(ctx, schedulerID) / c.DeleteSchedulers(ctx, schedulerIDs)

// Policies & technical config (read-only)
c.GetActivePoliciesByType(ctx, policyType)
c.GetAllActivePolicies(ctx)
c.GetTechnicalConfigs(ctx, scope, scopeTarget)
```

See [CONNECTOR_GO.md](CONNECTOR_GO.md) for the full reference.

### `livedata` — telemetry/detection streaming + live-stream/camera control

```go
ld := livedata.New(conn)   // dial live-data-service, default port 8003

stream, err := ld.StreamTelemetry(ctx, sn, frequencyMs, durationSeconds)
// stream is the raw grpc.ServerStreamingClient[...] — call stream.Recv() in a loop yourself.

ld.StreamDetections(ctx, sn)
ld.StreamNotifications(ctx, sn, eventTypes)

ld.StartLiveStream(ctx, req) / ld.StopLiveStream(ctx, req)
ld.ChangeLens(ctx, req) / ld.ChangeZoom(ctx, req)
ld.StartRecording(ctx, sn) / ld.StopRecording(ctx, sn)
ld.CapturePhoto(ctx, sn)
```

**No SDK-managed auto-reconnect** — unlike the Java SDK's `LiveData.streamTelemetryData` (which
manages reconnection with capped exponential backoff internally), the Go SDK hands back the raw
gRPC streaming client and leaves redialing a broken stream to the caller.

## What this SDK deliberately does *not* do

- **No unified auto-configured client.** There's no Go equivalent of Java's CDI-injected
  `ZequentClient` — dial each service's `grpc.ClientConnInterface` yourself and construct
  `connector.New(conn)` / `remotecontrol.New(conn)` / etc.
- **No built-in retry, circuit breaker, load balancing, or Stork service discovery.** The Java SDK's
  resilience layer (see [CONFIGURATION.md](CONFIGURATION.md)) isn't mirrored here — wrap calls with
  your own retry/backoff (e.g. `google.golang.org/grpc/backoff`) if you need it, or dial through
  whatever service mesh/proxy your deployment already uses.
- **No auto-reconnecting streams** (see `livedata` above).
- **The deprecated Mission/Task API.** Like the Java SDK, `MissionAutonomyService`'s old
  Mission/Task methods aren't backed by any RPC in the current proto contract — there was nothing
  working to mirror, so this package only has the current Application/Skill surface.

## Troubleshooting

**`verifying module: 404 Not Found`** — run the two `go env -w` commands in Step 1; the module lives
in a private GitHub org and won't resolve through the public Go proxy/sumdb otherwise.

**`fatal: could not read Username`** — make sure you have access to the repository and are
authenticated with GitHub (SSH key or a PAT in your git credential helper).

## Summary

1. `go get github.com/Zequent/zqnt-client-sdk-go@latest`
2. Dial the backend service(s) you need with a plain `grpc.NewClient(...)`.
3. Wrap the connection: `remotecontrol.New(conn)`, `connector.New(conn)`, etc.
4. Call methods directly — `ctx` in, `(*Response, error)` out.
