# Edge SDK (Go) — Quickstart

## Step 1: Add the module

```bash
go env -w GONOSUMDB="github.com/Zequent/*"
go env -w GONOPROXY="github.com/Zequent/*"
go get github.com/Zequent/zqnt-edge-sdk-go@latest
```

If you hit `verifying module: 404 Not Found`, you skipped the two `go env -w` lines above — the
module lives in a private GitHub org and won't resolve through the public proxy/sumdb otherwise. If
you hit `fatal: could not read Username`, make sure you're authenticated with GitHub (SSH key or a
PAT in your git credential helper) and have access to the repo.

## Step 2: Implement your adapter

Embed `adapter.UnimplementedEdgeAdapter` and override only the commands your hardware actually
supports — everything else automatically returns `NOT_IMPLEMENTED`. See
[Edge Adapter](edge-sdk-go-adapter.md) for the full command list.

```go
package main

import (
    "context"
    "net"

    edgesdk "github.com/Zequent/zqnt-edge-sdk-go"
    "github.com/Zequent/zqnt-edge-sdk-go/adapter"
    "github.com/Zequent/zqnt-edge-sdk-go/adapter/domains"
)

type MyDroneAdapter struct {
    adapter.UnimplementedEdgeAdapter
}

func (a *MyDroneAdapter) TakeOff(ctx context.Context, req *domains.TakeOffRequest) (*domains.CommandResult, error) {
    // send takeoff command to your hardware here
    return domains.SuccessWithTID("ok", req.TID, req.SN), nil
}

func (a *MyDroneAdapter) ReturnToHome(ctx context.Context, req *domains.ReturnToHomeRequest) (*domains.CommandResult, error) {
    // send return-to-home command to your hardware here
    return domains.SuccessWithTID("ok", req.TID, req.SN), nil
}

func main() {
    client, err := edgesdk.NewEdgeClient(
        "your-backend:50051", // Zequent backend address
        "YOUR-DEVICE-SN",     // device serial number
        &MyDroneAdapter{},
    )
    if err != nil {
        panic(err)
    }

    lis, err := net.Listen("tcp", ":9090")
    if err != nil {
        panic(err)
    }
    if err := client.StartServing(context.Background(), lis); err != nil {
        panic(err)
    }
}
```

**This single-address form only works if something in front of the platform multiplexes Connector,
Live Data, and Mission Autonomy onto one address/port** — in every real deployment topology in this
ecosystem, those are three independent Quarkus services on three independent ports (confirmed
directly in the SDK's own `config.go` doc comments). For a real deployment, pass
`WithConnectorAddr`/`WithLiveDataAddr`/`WithMissionAutonomyAddr` explicitly — see
[Configuration options](#configuration-options) below. Leaving them unset and pointing
`backendAddr` at just one of the three services will silently fail to reach the other two.

## Step 3: Run it

```bash
BACKEND_ADDR=your-backend:50051 DEVICE_SN=YOUR-SN go run main.go
```

See [`example/main.go`](https://github.com/Zequent/zqnt-edge-sdk-go/blob/main/example/main.go) in
the repo for a complete working example with graceful shutdown and logging.

## Configuration options

`NewEdgeClient` takes functional options as its last arguments:

```go
client, err := edgesdk.NewEdgeClient(
    backendAddr,
    deviceSN,
    &MyDroneAdapter{},
    edgesdk.WithLogger(myLogger),               // custom *slog.Logger
    edgesdk.WithTimeout(10 * time.Second),      // per-call timeout
    edgesdk.WithMaxRetries(3),                  // retry attempts for backend calls
    edgesdk.WithAssetType("ASSET_TYPE_DOCK"),
    edgesdk.WithAssetVendor("DJI"),
    edgesdk.WithAssetID("your-platform-asset-id"),
    edgesdk.WithConnectorAddr("connector-service:8010"),         // see note below
    edgesdk.WithLiveDataAddr("live-data-service:8003"),          // see note below
    edgesdk.WithMissionAutonomyAddr("mission-autonomy-service:8004"), // see note below
)
```

**`WithConnectorAddr`/`WithLiveDataAddr`/`WithMissionAutonomyAddr` matter more than the rest of
this list.** `backendAddr` alone only reaches whichever one service happens to be listening at that
address — Connector, Live Data, and Mission Autonomy are three independent Quarkus services on
three independent ports in every real deployment topology in this ecosystem (confirmed directly in
the SDK's own `config.go` doc comments). Set all three explicitly for any deployment that isn't
proxying all three services onto one address; leaving them unset is easy to miss since it fails
silently for whichever two services `backendAddr` doesn't happen to reach, rather than erroring at
startup.

The backend connection uses insecure gRPC credentials by default — wrap with your own
`grpc.WithTransportCredentials` (via a lower-level constructor, or by dialing yourself) for TLS in
production.

## Graceful shutdown

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
client.Shutdown(ctx) // stops the gRPC server, closes telemetry streams, closes the backend connection
```

## Producing telemetry

`client.LiveData()` gives you the telemetry-streaming client — see
[Backend Services](edge-sdk-go-backend-services.md#live-data) for the full interface.

```go
client.LiveData().ProduceTelemetryData(ctx, &domains.TelemetryRequestData{
    SN: "YOUR-DEVICE-SN",
    // ... position, battery, etc.
})
```
