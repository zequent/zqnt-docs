# Zequent Client SDK - Quick Start Guide (Go)

## For Customers: Using the SDK in Your Go Project

This guide shows you how to use the Zequent Client SDK in a Go application. See
[QUICKSTART.md](QUICKSTART.md) / [QUICKSTART_PYTHON.md](QUICKSTART_PYTHON.md) for the Java/Python
equivalents — the underlying gRPC contract is identical, only the calling convention differs.

## Step 1: Add the module

```bash
go env -w GONOSUMDB="github.com/Zequent/*"
go env -w GONOPROXY="github.com/Zequent/*"
go get github.com/Zequent/zqnt-client-sdk-go/v2@v2.0.0
```

Requires Go 1.26+ (this module's own `go.mod` pins `go 1.26.2`). The module path ends in `/v2`, and
so does every import path below — without it, Go installs the end-of-life 1.3 line.

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

    "github.com/Zequent/zqnt-client-sdk-go/v2/auth"
    "github.com/Zequent/zqnt-client-sdk-go/v2/remotecontrol"
)

// auth.DialOptions sends the client credential on every call; "" reads ZQNT_CLIENT_TOKEN.
opts := append(auth.DialOptions(""), grpc.WithTransportCredentials(insecure.NewCredentials()))
conn, err := grpc.NewClient("localhost:8002", opts...)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

rc := remotecontrol.New(conn)
```

The **client credential** is issued in the Admin Console under **Manage → Access & Integrations →
Credentials**. Without one the platform refuses every call. For TLS in production, pass real
transport credentials instead of `insecure.NewCredentials()`.

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

    "github.com/Zequent/zqnt-client-sdk-go/v2/auth"
    "github.com/Zequent/zqnt-client-sdk-go/v2/remotecontrol"
    devicecontrol "github.com/Zequent/zqnt-client-sdk-go/v2/gen/devicecontrol/contracts/proto"
)

func main() {
    opts := append(auth.DialOptions(""), grpc.WithTransportCredentials(insecure.NewCredentials()))
    conn, err := grpc.NewClient("localhost:8002", opts...)
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
rc.GoTo(ctx, sn, coordinate) / rc.GoToWithOptions(ctx, sn, coordinate, options)
rc.ReturnToHome(ctx, sn, altitude)
rc.EnterManualControl(ctx, sn, clientID, userID, sessionID)
rc.LookAt(ctx, sn, coordinate, locked)
rc.SendCustomCommand(ctx, sn, commandID, params, target)
rc.OpenCover(ctx, sn) / rc.CloseCover(ctx, sn, force)
rc.StartCharging(ctx, sn) / rc.StopCharging(ctx, sn)
rc.RebootAsset(ctx, sn)
rc.GetAssetRuntime(ctx, sn, assetID)
rc.GetCapabilities(ctx, sn)
```

Shares its request/response shapes with `EdgeAdapterService` (`devicecontrol` package) — the same
types flow end to end from this client through to the edge adapter that actually talks to the
hardware. See the [Remote Control API Reference](../api-reference/client-sdk-remote-control-go.md)
for every method (this list is illustrative, not exhaustive).

### `missionautonomy` — Applications & Skill executions

```go
ma := missionautonomy.New(conn)   // dial mission-autonomy-service, default port 8004

// Run a Skill of a deployed Application ("" version = Production, or the newest), or one command
ma.ExecuteApplication(ctx, sn, applicationID, skillID, "", params, idempotencyKey)
ma.ExecuteSimple(ctx, sn, commandID, params, idempotencyKey)

ma.GetSkillExecution(ctx, executionID) / ma.ListSkillExecutions(ctx, query)
ma.PauseSkillExecution(ctx, executionID) / ma.ResumeSkillExecution(ctx, executionID)
ma.CancelSkillExecution(ctx, executionID)
ma.SignalSkillExecution(ctx, executionID, nodeID, eventType, data, approved)

ma.UpsertApplication(ctx, app, expectedRevision) / ma.GetApplication(ctx, applicationID, version)
ma.ListApplications(ctx, scope, enabledOnly, pageSize, pageToken)
```

See [Applications & Skills](../concepts/applications-and-skills.md#go) for a runnable example, and
[Waypoint Missions](WAYPOINT_MISSIONS.md) for flying a waypoint route.

### `connector` — assets, schedulers, policies, config

```go
c := connector.New(conn)   // dial connector-service, default port 8010

c.GetAssetBySn(ctx, sn)

// Schedulers
c.ListSchedulers(ctx)
c.GetScheduler(ctx, schedulerID)
c.CreateScheduler(ctx, scheduler) / c.CreateSchedulers(ctx, schedulers)
c.UpdateScheduler(ctx, schedulerID, scheduler)
c.DeleteScheduler(ctx, schedulerID) / c.DeleteSchedulers(ctx, schedulerIDs)

// Skill Registry
c.ListSkillContracts(ctx, status, commandID)

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

## Troubleshooting

**`verifying module: 404 Not Found`** — run the two `go env -w` commands in Step 1; the module lives
in a private GitHub org and won't resolve through the public Go proxy/sumdb otherwise.

**`fatal: could not read Username`** — make sure you have access to the repository and are
authenticated with GitHub (SSH key or a PAT in your git credential helper).

## Summary

1. `go get github.com/Zequent/zqnt-client-sdk-go/v2@v2.0.0`
2. Dial the backend service(s) you need with `grpc.NewClient(...)` and `auth.DialOptions(...)` for the client credential.
3. Wrap the connection: `remotecontrol.New(conn)`, `connector.New(conn)`, etc.
4. Call methods directly — `ctx` in, `(*Response, error)` out.
