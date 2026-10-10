# Zequent Client SDK (Go) — Commands (3.0 preview)

> **Preview.** This page describes the Zequent **3.0** command and manual-control API (package
> `control`) and telemetry API (package `telemetry`) of the Go Client SDK. 3.0 is not released yet: the API below is available from a preview branch and needs a
> platform that runs 3.0. Names may still change before the release. For the current 2.0 line, see
> the [Go Quickstart](QUICKSTART_GO.md).

On a 3.0 platform every asset is commanded the same way: ask the asset what it can do, then run a
command by its dotted **command id** (`flight.takeoff`, `navigation.go_to`, `dock.open_cover`, …)
with a JSON object of params. There is no typed method per command, so a command a new adapter adds
works without a new SDK release.

Java: [Commands (Java)](COMMANDS-3.0.md) · Python: [Commands (Python)](COMMANDS_PYTHON-3.0.md) ·
Upgrading an integration: [Upgrading to 3.0](../concepts/upgrading-to-3.0.md).

## Step 1: Add the preview module

The 3.0 Go Client SDK is a new major module, **`github.com/zequent/zqnt-client-sdk-go/v3`** — note
`/v3` and the lower-case owner. Its generated message types come from
`github.com/zequent/zqnt-utils-golang/v2/gen/...` (the 3.0 types under `gen/zqnt/<domain>/v3`). Both
modules are private: you need GitHub access to them.

```bash
go env -w GOPRIVATE="github.com/zequent/*"
# preview branch, not a release
go get github.com/zequent/zqnt-client-sdk-go/v3@feature/v3-agentic
```

Go resolves the branch to a pseudo-version; run the `go get` again to pick up later preview commits.
Move to the `v3.0.0` release once it is out; do not run a preview in production. Requires Go 1.26+.

## Step 2: Connect

`control` is a client of the 3.0 command service, which remote-control serves on the same host and
port as the 2.x service, with the same client credential. `config.FromEnv()` reads the variables
every Zequent client SDK reads (`REMOTE_CONTROL_SERVICE_HOST` / `_PORT` / `_USE_PLAINTEXT`, `ZQNT_CLIENT_TOKEN`; see
[Client SDK Configuration](CONFIGURATION.md)):

```go
import (
    "github.com/zequent/zqnt-client-sdk-go/v3/config"
    "github.com/zequent/zqnt-client-sdk-go/v3/control"
)

cfg, err := config.FromEnv()
if err != nil {
    log.Fatal(err)
}
conn, err := cfg.Dial(cfg.RemoteControl) // TLS unless _USE_PLAINTEXT, plus the client credential
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

assets := control.New(conn)
```

`control.New` takes any `grpc.ClientConnInterface`, so a connection you dial yourself with
`auth.DialOptions("")` works too.

## Step 3: Discover what an asset can do

```go
set, err := assets.ListCapabilities(ctx, "DRONE-1")
if err != nil {
    log.Fatal(err)
}
for _, capability := range set.GetCapabilities() {
    fmt.Println(capability.GetCommandId(), capability.GetSafety().GetRisk(), capability.GetDisplayName())
}
```

`ListCapabilities(ctx, sn, opts...)` returns the asset's `*capabilityv3.CapabilitySet`
(`github.com/zequent/zqnt-utils-golang/v2/gen/zqnt/capability/v3`): one `Capability` per command,
with

- `GetCommandId()` — the dotted id you execute, and `GetDisplayName()` / `GetDescription()`;
- `GetInputSchema()` / `GetOutputSchema()` — JSON Schemas of the params and of the result;
- `GetSafety().GetRisk()` — `COMMAND_RISK_OBSERVE`, `_ADJUST`, `_MOVE` or `_CRITICAL`
  (a command without a rating counts as CRITICAL);
- `GetState()` — whether the command is available right now, with `GetUnavailableReason()`;
- `GetTarget()` — which part of the asset executes it (the asset, a sub-asset, a payload, a component);
- `GetCompletion()` — `COMPLETION_MODE_ON_REPLY` (the reply is the outcome) or
  `COMPLETION_MODE_ASYNCHRONOUS` (the outcome follows as an event — takeoff, go-to, return home);
- `GetErrors()` / `GetEvents()` — the error codes and events the command declares.

When the asset's adapter cannot be reached, the platform answers with the last snapshot it kept and
`set.GetSnapshotState()` is `SNAPSHOT_STATE_STALE`. `control.WithTarget(...)` lists the
capabilities of one part of the asset.

The capability's input schema is authoritative for an asset. The built-in commands every Zequent
adapter shares are listed in [Upgrading to 3.0](../concepts/upgrading-to-3.0.md#typed-call--command-id).

## Step 4: Execute a command by id

```go
result, err := assets.ExecuteCommand(ctx, "DRONE-1", "navigation.go_to",
    control.Params{"latitude": 47.7760, "longitude": 9.2671, "altitude": 60})
if err != nil {
    log.Fatal(err)
}
fmt.Println(result.GetState(), result.GetCommandExecutionId())
```

`ExecuteCommand(ctx, sn, commandID, params, opts...)` returns `(*capabilityv3.CommandResult, error)`:

- a command that finishes with the reply comes back `COMMAND_STATE_SUCCEEDED` with
  `GetResult()`;
- a command that takes time comes back `COMMAND_STATE_ACCEPTED` or `COMMAND_STATE_RUNNING` with a
  `GetCommandExecutionId()`; its outcome follows as events (Step 6) — or use `ExecuteAndWait` (Step 5);
- a command that was `REJECTED`, or already ended `FAILED`, `CANCELLED` or `TIMED_OUT`, is returned
  as a `*control.CommandError` (see [Errors](#errors)). This differs from the Java and Python SDKs,
  which return a `FAILED` run as a result.

`control.Params` is a `map[string]any` sent as a `google.protobuf.Struct`; values must be what
`encoding/json` would produce (`nil`, `bool`, numbers, `string`, `[]any`, `map[string]any`) — other
values are refused before anything is sent. `nil` params send none. **Leave a param out when you
have no value**: the platform reads a missing coordinate as "not given", while `0` means latitude 0 /
longitude 0 / ground level. `navigation.go_to`'s `altitude` is metres above the **takeoff point**
(left out = 40 m).

### Options

```go
result, err := assets.ExecuteCommand(ctx, "DRONE-1", "flight.return_to_home",
    control.Params{"altitude": 80},
    control.WithTimeout(10*time.Minute),
    control.WithReason("battery low"),
    control.WithIdempotencyKey(myRequestID))
```

| Option | Meaning |
| --- | --- |
| `WithTarget(kind capabilityv3.TargetType, ref string)` | A sub-asset, payload or component instead of the asset itself |
| `WithTimeout(d time.Duration)` | Replaces the capability's own default timeout |
| `WithReason(reason string)` | A human reason; **required** for `COMMAND_RISK_CRITICAL` commands |
| `WithNoFlyZoneOverride()` | Fly through no-fly zones instead of around them; honoured for an organization admin or a system admin only, anybody else's command is refused |
| `WithIdempotencyKey(key string)` | A retried call with the same key returns the first call's result |
| `WithRequestID(id string)` | Correlates the call with the platform's logs and events; generated otherwise |

## Step 5: Execute and wait for the outcome

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Minute)
defer cancel()

result, err := assets.ExecuteAndWait(ctx, "DRONE-1", "flight.takeoff",
    control.Params{"latitude": 47.7752, "longitude": 9.2658, "altitude": 40})
var refused *control.CommandError
if errors.As(err, &refused) {
    log.Fatalf("%v (category %s, retryable %t)", refused, refused.Category(), refused.Retryable())
}
if err != nil {
    log.Fatal(err)
}
fmt.Println(result.GetCommandExecutionId(), result.GetState(), result.GetResult().AsMap())
```

`ExecuteAndWait(ctx, sn, commandID, params, opts...)` is `ExecuteCommand` followed by waiting for
that run's outcome:

- It opens a watch on the asset's command events (`WatchAsset`) **before** it sends the command, so
  an outcome that arrives quickly is not missed.
- It follows **only the execution id** this call started; events of other runs on the same asset
  are ignored.
- **Only `COMMAND_STATE_SUCCEEDED` is success**: it returns that result with a `nil` error. A run
  that ends `REJECTED`, `FAILED`, `CANCELLED` or `TIMED_OUT` is a `*control.CommandError` carrying
  the final `Result`.
- `ctx` bounds the wait. When it ends first, the call returns an error and **the command keeps
  running** on the platform — `CancelCommand` stops it. A watch that ends before the run does
  returns an error wrapping `io.ErrUnexpectedEOF` (or the gRPC error that ended it).

## Step 6: Watch command events

```go
events, err := assets.WatchCommand(ctx, result.GetCommandExecutionId())
if err != nil {
    log.Fatal(err)
}
for {
    event, err := events.Recv()
    if errors.Is(err, io.EOF) {
        break // the run's final event was delivered
    }
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(event.GetState(), event.GetProgress(), event.GetMessage())
}
```

- `WatchCommand(ctx, commandExecutionID)` — events of one run **from now on**; the stream ends with
  `io.EOF` after its final event.
- `WatchAsset(ctx, sn)` — events of every command on the asset from now on, until `ctx` ends.

Both return a `*control.Events`; `Recv()` returns the next `*capabilityv3.CommandEvent`, `io.EOF`
when the stream has ended, or the gRPC error that ended it. Each event carries
`GetCommandExecutionId()`, `GetCommandId()`, `GetState()`, `GetProgress()` (0..1, when the adapter
can tell), `GetRemaining()`, `GetMessage()`, and on the final event `GetResult()` / `GetError()`.
`control.Terminal(state)` reports whether a state is final.

A watch opened after `ExecuteCommand` returns can miss the end of a fast run. To wait for a run's
outcome, use `ExecuteAndWait`.

## Step 7: Cancel a run

```go
cancelled, err := assets.CancelCommand(ctx, result.GetCommandExecutionId(), "operator abort")
```

`CancelCommand(ctx, commandExecutionID, reason, opts...)` returns the run's `CommandResult`; a cancel
that comes back `REJECTED` or `FAILED` is a `*control.CommandError`.

## Errors

- **A command the platform or the asset did not carry out** — `REJECTED` before it started (unknown
  id, params, precondition, permission), or `FAILED`, `CANCELLED`, `TIMED_OUT` — is a
  `*control.CommandError`. Get it with `errors.As`. It has `State()`, `Category()` (a
  `commonv3.ErrorCategory`: `ERROR_CATEGORY_INVALID_ARGUMENT`, `_PERMISSION_DENIED`,
  `_PRECONDITION_FAILED`, `_ASSET`, `_SERVICE`, `_TIMEOUT`, `_NOT_FOUND`), `Retryable()`, and the full
  `Result` (`Result.GetError().GetCode()` is the stable code, such as `command.invalid_params` or
  `flight.not_airborne`).
- **A call that never got that far keeps its gRPC status**: `status.Code(err)` is
  `codes.Unauthenticated`, `codes.PermissionDenied`, `codes.InvalidArgument`, `codes.Unavailable`, …
  On a connection dialed with `auth.DialOptions` (as `cfg.Dial` does), an `Unauthenticated` or
  `PermissionDenied` refusal comes back with a message that says what to do; the code is unchanged.

`result.GetResult().AsMap()` turns a result or event payload into a `map[string]any`.

The Go SDK has no built-in retry or circuit breaker (as in 2.0); retry with `WithIdempotencyKey`
so a retried command runs once.

## Manual control (stick input)

Stick input goes over one client-streaming session per asset. Put the asset in manual control
first, then send inputs (roll, pitch, yaw, throttle from -1 to 1, optionally a gimbal pitch) at your
control rate:

```go
import (
    edgev3 "github.com/zequent/zqnt-utils-golang/v2/gen/zqnt/edge/v3"
    "google.golang.org/protobuf/proto"
)

if _, err := assets.ExecuteAndWait(ctx, "DRONE-1", "flight.manual.enter", nil); err != nil {
    log.Fatal(err)
}
stick, err := assets.StreamManualControl(ctx, "DRONE-1")
if err != nil {
    log.Fatal(err)
}
err = stick.Send(&edgev3.ManualControlInput{Pitch: 0.2, Throttle: 0.5})
err = stick.Send(&edgev3.ManualControlInput{Throttle: 0.5, GimbalPitch: proto.Float32(-30)})
accepted, err := stick.Close()
```

- `StreamManualControl(ctx, sn)` returns a `*control.ManualControl`. One session controls one asset;
  the first input names it. Inputs go out in the order they are sent until `Close`, or until `ctx`
  ends, which aborts the stream.
- `Send(*edgev3.ManualControlInput)` sends one input. `Send` and `Close` may be called from different
  goroutines.
- `Close()` ends the session and returns `(int64, error)`: the number of inputs the platform passed
  on to the asset. An error in the platform's answer is a `*control.CommandError`; calling `Close`
  again returns the same answer.
- When the platform refuses the session (no adapter reachable for the asset, an asset of another
  organization) or the adapter drops it, `Send` and `Close` return that error with its gRPC status
  (`status.Code(err)`). `Send` after `Close` returns `control.ErrManualControlClosed`.

The 2.x stream, `remotecontrol.Client.OpenManualControlInputStream`, is in the deprecated
`remotecontrol` package and stays for 2.x platforms.

## Live telemetry

Package `telemetry` watches 3.0 telemetry from the live-data service (`LIVE_DATA_SERVICE_*`, same
client credential):

```go
import "github.com/zequent/zqnt-client-sdk-go/v3/telemetry"

conn, err := cfg.Dial(cfg.LiveData)
if err != nil {
    log.Fatal(err)
}
watch, err := telemetry.New(conn).WatchTelemetry(ctx, []string{"DOCK-1"}, telemetry.WithMinInterval(time.Second))
if err != nil {
    log.Fatal(err)
}
defer watch.Close()
for {
    item, err := watch.Recv()
    if err != nil {
        log.Fatal(err) // io.EOF, telemetry.ErrSilent or the gRPC error that ended the watch
    }
    if sample := item.GetSample(); sample != nil {
        fmt.Println(sample.GetAsset().GetSn(), sample.GetPosition(), sample.GetBatteryPercent(), sample.GetDetails().AsMap())
    } else {
        fmt.Println(item.GetSourceStatus().GetAsset().GetSn(), item.GetSourceStatus().GetState())
    }
}
```

`telemetry.New(conn).WatchTelemetry(ctx, sns, opts...)` returns a `*telemetry.Watch`; its `Recv()`
returns the next `*telemetryv3.WatchTelemetryResponse` with either `GetSample()` or
`GetSourceStatus()` set (heartbeats are skipped):

- **`TelemetrySample`** (`github.com/zequent/zqnt-utils-golang/v2/gen/zqnt/telemetry/v3`): the fields
  every asset shares — `GetPosition()`, `GetRelativeAltitude()`, `GetHeadingDegrees()`,
  `GetHorizontalSpeed()`, `GetVerticalSpeed()`, `GetBatteryPercent()`, `GetObservedAt()` — and
  everything device-specific in `GetDetails()`, under the keys the asset declares in its
  capabilities' `telemetry_fields`;
- **`SourceStatus`**: every watched asset's state first, then each change to `SOURCE_STATE_ONLINE`,
  `SOURCE_STATE_STALE` or `SOURCE_STATE_NO_DATA` (with `GetLastSampleAt()`).

No asset (`nil` or empty `sns`) watches every asset of the caller's organization; a dock's serial
also brings the samples of its drone. Options: `telemetry.WithMinInterval(d)` keeps at most one
sample per asset per interval (without it every sample comes); `telemetry.WithHeartbeatTimeout(d)`
replaces `telemetry.DefaultHeartbeatTimeout` (**35 s**).

The platform sends heartbeats on a quiet stream: when neither a sample nor a heartbeat arrives
within the heartbeat timeout, `Recv` returns `telemetry.ErrSilent`. It returns `io.EOF` when the
platform ended the watch, or the gRPC error that ended it. `Close()` (or ending `ctx`) ends the
watch. **A watch does not reconnect**: watch again.

`livedata.Client.StreamTelemetry` stays the 2.x telemetry stream.

## Against a 2.x platform

Packages `control` and `telemetry` need a 3.0 platform. A 2.x platform does not serve the 3.0
command and telemetry services: the calls fail with `status.Code(err) == codes.Unimplemented`. The SDK does **not** switch to the 2.x
calls on its own. Against 2.x, keep using package `remotecontrol` (still in the `/v3` module and
marked deprecated) and package `livedata`. See [Upgrading to 3.0](../concepts/upgrading-to-3.0.md#running-against-2x-platforms).

## Runnable example

The SDK repository's `examples/command` lists an asset's capabilities and runs one command, waiting
for its outcome:

```bash
ZQNT_CLIENT_TOKEN=... go run ./examples/command DRONE-1 flight.takeoff '{"altitude": 40}'
```
