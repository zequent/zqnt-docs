# Zequent Edge SDK (Go) — Adapters on 3.0 (preview)

> **Preview.** This page describes the **3.0** adapter API of the Go Edge SDK. 3.0 is not released
> yet: the API below is available as a preview. Names may still change before the release. For the
> current 2.0 line, see [Go Edge Adapter](edge-sdk-go-adapter.md) and the
> [Go Quickstart](edge-sdk-go-quickstart.md).

On 3.0 an adapter registers each command **once** — dotted id, input and output JSON Schema,
handler. The advertised capabilities, the v3 `ExecuteCommand` the platform calls and the v2 typed
RPCs a 2.x platform still calls all come from that registration, so the same adapter works against
2.x and 3.0 platforms.

Java: [Adapters on 3.0 (Java)](edge-sdk-adapter-3.0.md) · Python:
[Adapters on 3.0 (Python)](edge-sdk-python-adapter-3.0.md) · Platform:
[Upgrading the platform to 3.0](../concepts/upgrading-platform-to-3.0.md).

## Step 1: Add the preview module

The 3.0 SDK is a new major module, **`github.com/zequent/zqnt-edge-sdk-go/v3`** (`/v3`, lower-case
owner). Its generated types and the telemetry mapping come from the public module
**`github.com/zequent/zqnt-api-go/v3`** (Apache 2.0), which replaces `zqnt-utils-golang`; the
packages below `gen/` keep their paths.

```bash
# preview branch, not a release
go get github.com/zequent/zqnt-edge-sdk-go/v3@feature/public-api-split
```

Go resolves the branch to a pseudo-version; run `go get` again to pick up later preview commits and
move to `v3.0.0` once it is out — do not run a preview in production. Requires Go 1.25+.

Moving an adapter from 2.x:

```bash
grep -rl zqnt-edge-sdk-go/v2 --include='*.go' . | xargs sed -i \
  's#github.com/Zequent/zqnt-edge-sdk-go/v2#github.com/zequent/zqnt-edge-sdk-go/v3#g'
go get github.com/zequent/zqnt-edge-sdk-go/v3@feature/public-api-split && go mod tidy
```

Imports of generated types move to `github.com/zequent/zqnt-api-go/v3/gen/...`. Authentication
(`ZQNT_EDGE_TOKEN` / `WithEdgeToken`, `ZQNT_PLATFORM_PUBLIC_KEY` / `WithPlatformPublicKey`,
`ZQNT_EDGE_AUTH_DISABLED` / `WithoutPlatformAuth()`) is unchanged from 2.0.

## Step 2: Declare the commands

Embed `adapter.Base` and register the commands:

```go
package main

import (
    "context"

    "github.com/zequent/zqnt-edge-sdk-go/v3/adapter"
    "github.com/zequent/zqnt-edge-sdk-go/v3/adapter/domains"
)

var altitudeInput = map[string]any{
    "type":       "object",
    "properties": map[string]any{"altitude": map[string]any{"type": "number", "minimum": 2}},
}

type Drone struct {
    adapter.Base
    events adapter.CommandEventPublisher
}

func newDrone() *Drone {
    d := &Drone{}
    d.SetAssetType("ASSET_TYPE_DRONE")
    d.MustRegisterCommand("flight.takeoff", altitudeInput, nil, d.takeOff,
        adapter.WithDescription("Take off and climb to the given altitude above the takeoff point"),
        adapter.WithCompletion(domains.CompletionAsynchronous, "flight.takeoff.completed"))
    if gear, ok := domains.StandardTelemetryField("drone.gear"); ok {
        d.DeclareTelemetryField(gear)
    }
    d.DeclareTelemetryField(domains.TelemetryField{
        Key: "vendor.acme.tank_level", Type: domains.TelemetryValueNumber, Unit: "%"})
    return d
}

func (d *Drone) takeOff(ctx context.Context, req *domains.CustomCommandRequest) (*domains.CommandResult, error) {
    altitude, _ := req.Params["altitude"].(float64) // validated against the input schema
    // start the climb on the device, report the outcome later (Step 4)
    _ = altitude
    return domains.Success("climbing", req.SN), nil
}
```

- **Ids.** Reuse the platform's ids for shared actions (`flight.takeoff`, `navigation.go_to`,
  `dock.open_cover`, …); a device-specific command goes under `vendor.<vendor>.<name>`.
- **Handler.** `adapter.CommandHandler` is `func(ctx, *domains.CustomCommandRequest)
  (*domains.CommandResult, error)`. The request carries `SN`, `CommandID`, `TargetRef`, `Params` and
  `CommandExecutionID` (the platform's id of this run on v3; empty over v2, where `TID` carries it).
- **Validation.** Params are validated against the input schema before the handler runs; whole
  numbers under an `integer` property arrive as `int64`, every other number as `float64`. A mismatch
  is `REJECTED` with error code `command.invalid_params` and a message naming each field. A schema
  that does not compile fails `RegisterCommand` (`MustRegisterCommand` panics).
- **Result.** `domains.Success(...)` / `SuccessWithOutput(...)`, `domains.Accepted(message,
  externalExecutionID, sn)`, `domains.Rejected(code, message, sn)`, `domains.Error(...)`. A success is
  reported `ACCEPTED` on v3 — the outcome follows as an event — when it carries an external execution
  id or the command declares `CompletionAsynchronous`; otherwise `SUCCEEDED`.
- **Options.** `WithDescription`, `WithCompletion(mode, completionEvent)`, `WithTarget(type, ref)`,
  `Unavailable(reason)`, `WithCapability(func(*domains.Capability))` for any other field.
- **Changing what the asset can do.** `RegisterCommand` again replaces an id, `UnregisterCommand(id)`
  withdraws it; every registry change is reported to the platform.

Adapters written against the typed methods (`adapter.UnimplementedEdgeAdapter` + `TakeOff`, …) keep
working — the v3 server runs a built-in id through the typed method that implements it — but those
methods are deprecated in favour of the registry.

### Command risk (`CommandSafety`)

The 3.0 contract rates every command with a `CommandSafety` (`COMMAND_RISK_OBSERVE`, `_ADJUST`,
`_MOVE`, `_CRITICAL`, plus reversibility and an undo command). The 3.0 SDK previews **do not let an
adapter set it**: the platform rates a command by its command catalog, and an id outside the
catalog — every `vendor.*` command — counts as **critical** (a client must give a reason to run it).

## Step 3: Serve and connect

```go
client, err := edgesdk.NewEdgeClient(backendAddr, deviceSN, drone,
    edgesdk.WithRemoteControlAddr(remoteControlAddr))
if err != nil {
    return err
}
drone.events = client.LiveData()

lis, _ := net.Listen("tcp", ":9090")
return client.StartServing(ctx, lis)
```

`StartServing` serves the v2 and the v3 `EdgeAdapterService` from the same adapter and reports the
capabilities. remote-control serves `EdgeGatewayService` (capability reports, v3 command events):
point the SDK at it with `WithRemoteControlAddr` unless the main endpoint multiplexes it
(`WithConnectorAddr`, `WithLiveDataAddr`, `WithMissionAutonomyAddr` do the same for the other
services).

| What | 3.0 platform | 2.x platform (answers `UNIMPLEMENTED`) |
| --- | --- | --- |
| Commands from the platform | v3 `ExecuteCommand` | v2 typed RPCs, served alongside |
| Capabilities (on start, on every registry change, `client.RefreshCapabilities()`) | `ReportCapabilities`, with the declared telemetry fields | v2 `ReportAssetRuntime` |
| Command events (`LiveData().PublishCommandExecutionEvent`) | `PublishCommandEvent` under the platform's `command_execution_id` | v2 notification stream |
| `LiveData().PublishTelemetrySample` | `TelemetryIngestService.PublishTelemetry` | v2 `ProduceTelemetry`, catalog keys mapped — see Step 5 |
| `LiveData().PublishDetections` | `TelemetryIngestService.PublishDetections` | v2 `ProduceDetection` |
| `LiveData().PublishAlert` | `TelemetryIngestService.PublishAlerts` | dropped (no v2 counterpart) |

**Automatic fallback.** After `UNIMPLEMENTED` the SDK stays on v2 for **10 minutes**, then tries v3
again — an adapter picks up a platform upgrade without a restart. The first item sent to a v2-only
platform can be lost before the SDK switches.

An adapter that serves more assets than its own `SN` (a fleet behind one endpoint) adds them with
`client.TrackCapabilities(sn)` (`UntrackCapabilities(sn)` to stop).

## Step 4: Report command events

```go
err := d.events.PublishCommandExecutionEvent(ctx, &domains.CommandExecutionEvent{
    SN:                 req.SN,
    CommandExecutionID: req.CommandExecutionID,
    ExternalExecutionID: req.TID,
    CommandID:          req.CommandID,
    Status:             domains.CommandExecutionSucceeded,
    Output:             map[string]any{"altitude": 40.0},
})
```

On v3 the event goes to `PublishCommandEvent` under `CommandExecutionID` (it takes precedence over
`ExternalExecutionID`); on v2 over the notification stream. `OccurredAt` defaults to the publish
time. `Progress` (0..1) belongs on a `CommandExecutionRunning` event; finish every accepted command
with exactly one terminal status (`CommandExecutionSucceeded`, `…Failed` with `ErrorCode`,
`…Cancelled`). Publishing through the `adapter.CommandEventPublisher` interface lets the conformance
kit's `Recorder` stand in for the client in tests.

## Step 5: Publish telemetry

```go
battery := 81.0
err := client.LiveData().PublishTelemetrySample(ctx, &domains.TelemetrySample{
    SN:             deviceSN,
    Position:       &domains.GeoPoint{Lat: 47.5, Lon: 9.7},
    BatteryPercent: &battery,
    Details:        map[string]any{"drone.gear": 1, "vendor.acme.tank_level": 63.0},
})
```

`TelemetrySample` has the fields every asset shares — `Position` (`Alt` above sea level),
`RelativeAltitude` (above the takeoff point), `HeadingDegrees`, `HorizontalSpeed`, `VerticalSpeed`,
`BatteryPercent` — and `Details` for everything device-specific. `ObservedAt` defaults to now.
`PublishDetections(ctx, *domains.DetectionBatch)` and `PublishAlert(ctx, *domains.Alert)` (severity
`domains.AlertSeverityInfo` / `Warning` / `Critical`) publish detections and alerts. The v3 streams
are long-lived and reconnect with backoff; an item sent while a stream waits for its next attempt
gets `livedata.ErrReconnecting`.

**Standard keys.** The platform's telemetry catalog
(`github.com/zequent/zqnt-api-go/v3/telemetry`) has 64 detail keys — listed in
[The telemetry catalog](edge-sdk-adapter-3.0.md#the-telemetry-catalog) — as constants
(`telemetry.DockCoverState`, `telemetry.DroneGear`, …) with `telemetry.Fields()` and
`telemetry.Field(key)`. `domains.StandardTelemetryField(key)` turns one into a `domains.TelemetryField`
with the platform's type, unit and allowed values (and reports `false` for a key outside the catalog).

**On a 2.x platform** a sample goes over v2 `ProduceTelemetry`: **catalog keys in `Details` land in
their v2 fields, other keys are dropped.** The v2 telemetry API (`ProduceTelemetryData`,
`ProduceTelemetry`) is unchanged.

## Pairing

The Go SDK has no claim-code helpers (no `ensureAsset` / `redeemAssetClaim` equivalent). Provision
Go-adapter assets in the console, or pair them with a claim code from a tool built on the Java or
Python SDK. The platform's organization-UUID bind is **planned** to be removed in 3.0.

## Step 6: Check the adapter with the conformance kit

```go
import "github.com/zequent/zqnt-edge-sdk-go/v3/conformance"

func TestConformance(t *testing.T) {
    recorder := &conformance.Recorder{}
    drone := newDrone()
    drone.events = recorder
    conformance.Run(t, drone, conformance.Options{Events: recorder})
}
```

`Run` reports one subtest per rule: schemas parse, advertised ids are executable, executable ids
are advertised, and an `ACCEPTED` command is completed by an event with `occurred_at` (within
`CompletionTimeout`, default 5 s). It **executes** commands — with `Options.Params` per id, or params
generated from the input schema — so run it against fakes or a simulator. `conformance.Check`
returns the `Report` instead of failing a test.
