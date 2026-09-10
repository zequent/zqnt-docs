# Zequent Edge SDK (Go)

The Go Edge SDK connects a physical asset (drone, dock, vehicle) to Zequent by running a small gRPC
server your process hosts — the platform dials in and calls your `EdgeAdapter` implementation, the
same inversion-of-control shape as the Java/Python Edge SDKs' `EdgeAdapterService`.

## Status: an older API surface than Java/Python — read this before you start

This SDK predates the platform's current Skill/Capability/Application model. Its command interface
(`EdgeAdapter`), and its `connector`/`missionautonomy` client packages, are still on the **old**
Mission/Task/Scheduler contract (`CreateMission`, `CreateTask`, `StartTask`, `GetCapabilities`'s
2-value `available bool` schema) — not the Skill Registry / Application-and-Skill graph model the
Java/Python Edge
SDK docs describe. It's real, tagged (`v1.0.0`/`v1.0.1`), published (`go get
github.com/Zequent/zqnt-edge-sdk-go@latest` works), and CI'd — just architecturally older than its
Java/Python counterparts.

**One piece has caught up**: a `skillregistry` package (self-report command contracts into the
platform's persisted Skill Registry, mirroring the Java/Python `ObserveSkillContract`/
`ListSkillContracts` surface) exists on branch `feature/skill-registry-v2`, **not yet merged to
main** — check whether it's landed before depending on it. It vendors its own up-to-date generated
proto code alongside (not replacing) the older `proto/` submodule the rest of the SDK still uses —
see [Backend Services](edge-sdk-go-backend-services.md#skill-registry-unmerged) for why, and what
that means for `GetCapabilities`'s wire format in the meantime.

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
