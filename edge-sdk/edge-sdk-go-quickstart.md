# Edge SDK (Go) — Quickstart

## Step 1: Add the module

```bash
go env -w GONOSUMDB="github.com/Zequent/*"
go env -w GONOPROXY="github.com/Zequent/*"
go get github.com/Zequent/zqnt-edge-sdk-go/v2@v2.0.0
```

The module path ends in `/v2`, and so does every import path below — without it, Go installs the
end-of-life 1.3 line. Go 1.25 or newer is required.

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

    edgesdk "github.com/Zequent/zqnt-edge-sdk-go/v2"
    "github.com/Zequent/zqnt-edge-sdk-go/v2/adapter"
    "github.com/Zequent/zqnt-edge-sdk-go/v2/adapter/domains"
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
    // ZQNT_EDGE_TOKEN and ZQNT_PLATFORM_PUBLIC_KEY are read from the environment
    client, err := edgesdk.NewEdgeClient(
        "live-data-service:8003", // main backend address
        "YOUR-DEVICE-SN",         // device serial number
        &MyDroneAdapter{},
        edgesdk.WithConnectorAddr("connector-service:8010"),
        edgesdk.WithLiveDataAddr("live-data-service:8003"),
        edgesdk.WithMissionAutonomyAddr("mission-autonomy-service:8004"),
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

Connector, Live Data and Mission Autonomy are three separate services on three ports, so pass all
three addresses with `WithConnectorAddr`/`WithLiveDataAddr`/`WithMissionAutonomyAddr` — see
[Configuration options](#configuration-options) below. The first argument alone reaches only
whichever service listens there, and the other two fail silently.

## Step 2b: Credentials

The SDK reads the adapter's credentials from the environment:

| Variable | Purpose |
| --- | --- |
| `ZQNT_EDGE_TOKEN` | The adapter's **edge credential**, sent on every call to the platform. Issued in the Admin Console under **Manage → Access & Integrations → Credentials** (kind *Edge adapter*) |
| `ZQNT_PLATFORM_PUBLIC_KEY` | The platform's public key (alias `SERVICE_AUTH_PUBLIC_KEY`). The adapter's server refuses every call the platform did not sign with it |
| `ZQNT_EDGE_AUTH_DISABLED` | `true` turns that check off — for a local stack only |

`WithEdgeToken`, `WithPlatformPublicKey` and `WithoutPlatformAuth` set the same from code. Health
probes stay open.

## Step 3: Run it

```bash
ZQNT_EDGE_TOKEN=<your edge credential> ZQNT_PLATFORM_PUBLIC_KEY=<the platform's public key> go run .
```

To call it by hand with `grpcurl`, run it with `ZQNT_EDGE_AUTH_DISABLED=true` — locally only. The
server has no gRPC reflection, so point `grpcurl` at the protocol files (`edge.proto` from the
`zqnt-protos` repository):

```bash
grpcurl -plaintext -import-path zqnt-protos -proto edge.proto -d '{
  "base": {"sn": "YOUR-DEVICE-SN", "tid": "test-123", "timestamp": "2026-01-01T00:00:00Z"},
  "coordinate": {"latitude": 47.3769, "longitude": 8.5417, "altitude": 100.0}
}' localhost:9090 zqnt.EdgeAdapterService/TakeOff
```

See [`example/main.go`](https://github.com/Zequent/zqnt-edge-sdk-go/blob/v2.0.0/example/main.go) in
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
    edgesdk.WithEdgeToken(token),               // default: ZQNT_EDGE_TOKEN
    edgesdk.WithPlatformPublicKey(publicKey),   // default: ZQNT_PLATFORM_PUBLIC_KEY
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

The connections to the platform use plaintext gRPC; there is no option for TLS. If your deployment
requires it, terminate TLS at a proxy in front of the platform services.

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
