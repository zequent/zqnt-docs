# Zequent Edge SDK (Go)

The Go Edge SDK connects a physical asset (drone, dock, vehicle) to Zequent by running a small gRPC
server your process hosts — the platform dials in and calls your `EdgeAdapter` implementation, the
same inversion-of-control shape as the Java/Python Edge SDKs' `EdgeAdapterService`.

## Status: an older API surface than Java/Python — read this before you start

This SDK has a narrower surface than its Java and Python counterparts. Its `connector` client
covers assets and organizations only — there is no task or mission lookup, so a Go adapter cannot
resolve a bare task id and can only take the command-based execution path (see
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md)). Its `missionautonomy` client exposes
scheduler lookup only. Its `Capability` struct is near-parity with Java's, though — confirmed
field-for-field against source, it's missing only the three JSON-Schema-shaped fields
(`constraints`, `inputSchema`, `outputSchema`); see
[Edge Adapter — Capability reporting](edge-sdk-go-adapter.md#capability-reporting).

It is real, tagged, published (`go get github.com/Zequent/zqnt-edge-sdk-go@latest` works) and CI'd
— just smaller in scope. Build a Go adapter on it when the command surface is enough for your
device; use the Java or Python Edge SDK when you need the task lifecycle. **No confirmed real Go
adapter exists in this ecosystem today** — every real, in-production adapter found (DJI, MAVLink,
SAPIENT, Betaflight, RNS) is a Java or Python implementation.

## Tech Specs

| Requirement | Version |
|-------------|---------|
| Go          | 1.24+   |
| Module      | `github.com/Zequent/zqnt-edge-sdk-go` |
| Transport   | gRPC (server you host; client stubs to the backend) |

## Overview

Your application acts as a gRPC **server** the Zequent backend connects to:

```
Zequent Backend  ──gRPC──>  Your App (EdgeAdapter)  ──>  Hardware
```

`edgesdk.NewEdgeClient` wires together three things in one call:

- An [`adapter.EdgeAdapter`](edge-sdk-go-adapter.md) implementation you provide — receives commands
  (TakeOff, GoTo, ReturnToHome, ...) and translates them to your hardware.
- A gRPC server exposing `EdgeAdapterService` for the platform to call into.
- Client stubs for `LiveData`, `Connector`, and `MissionAutonomy` (accessed via `client.LiveData()`
  / `client.Connector()` / `client.MissionAutonomy()`) — see
  [Backend Services](edge-sdk-go-backend-services.md).

## Available Documentation

| Document | Description |
|----------|--------------|
| [Quickstart](edge-sdk-go-quickstart.md) | Get a new adapter running in minutes |
| [Edge Adapter](edge-sdk-go-adapter.md) | The full `EdgeAdapter` command reference |
| [Backend Services](edge-sdk-go-backend-services.md) | Connector, Live Data, Mission Autonomy client packages |

## Quick Start

```bash
go env -w GONOSUMDB="github.com/Zequent/*"
go env -w GONOPROXY="github.com/Zequent/*"
go get github.com/Zequent/zqnt-edge-sdk-go@latest
```

```go
type MyDroneAdapter struct {
    adapter.UnimplementedEdgeAdapter   // only override what your hardware supports
}

func (a *MyDroneAdapter) TakeOff(ctx context.Context, req *domains.TakeOffRequest) (*domains.CommandResult, error) {
    return domains.SuccessWithTID("ok", req.TID, req.SN), nil
}

client, _ := edgesdk.NewEdgeClient("your-backend:50051", "YOUR-DEVICE-SN", &MyDroneAdapter{})
lis, _ := net.Listen("tcp", ":9090")
client.StartServing(context.Background(), lis)
```

For a complete walkthrough, see the [Quickstart Guide](edge-sdk-go-quickstart.md).
