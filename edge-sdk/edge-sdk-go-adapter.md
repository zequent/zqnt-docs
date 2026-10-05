# Edge SDK (Go) — Edge Adapter

The `adapter.EdgeAdapter` interface (`github.com/Zequent/zqnt-edge-sdk-go/v2/adapter`) is the core
contract of the Go Edge SDK — the direct hardware-command surface, one method per operation, plus
`SendCustomCommand` for everything else. See the [Overview](edge-sdk-go-overview.md) for what this SDK
covers.

## How it works

```
Zequent Backend  ──(gRPC)──>  adapter/grpc.Server  ──(delegates)──>  Your EdgeAdapter implementation
```

`edgesdk.NewEdgeClient` registers the gRPC server for you (see
[Quickstart](edge-sdk-go-quickstart.md)) — you never touch `adapter/grpc` directly, just implement
`EdgeAdapter` and hand it to `NewEdgeClient`.

## Implementing the interface

Embed `adapter.UnimplementedEdgeAdapter` in your concrete type to get `NOT_IMPLEMENTED` defaults for
every method, then override only what your hardware supports:

```go
type MyDroneAdapter struct {
    adapter.UnimplementedEdgeAdapter
    drone *hardware.Drone
}

func (a *MyDroneAdapter) TakeOff(ctx context.Context, req *domains.TakeOffRequest) (*domains.CommandResult, error) {
    if err := a.drone.TakeOff(req.Coordinates.Alt); err != nil {
        return domains.Error(err.Error(), req.SN), nil
    }
    return domains.SuccessWithTID("takeOff accepted", req.TID, req.SN), nil
}
```

Note the pattern: a hardware-side failure is returned as `domains.Error(...)` inside a **successful**
`(*CommandResult, nil)` return, not as a Go `error` — the `error` return is for transport/protocol
failures. Both are distinct from `NOT_IMPLEMENTED`, which `UnimplementedEdgeAdapter` already handles
for anything you don't override.

## Command reference

### Flight control
| Method | Description |
|--------|--------------|
| `TakeOff(ctx, *TakeOffRequest)` | Take off to a given coordinate/altitude |
| `GoTo(ctx, *GoToRequest)` | Fly to coordinates |
| `ReturnToHome(ctx, *ReturnToHomeRequest)` | Return to home position |

### Manual control
| Method | Description |
|--------|--------------|
| `EnterManualControl(ctx, sn)` | Enter manual RC control mode |
| `ExitManualControl(ctx, sn)` | Exit manual RC control mode |
| `ManualControlInput(ctx, *ManualControlInput)` | Deliver one frame of streaming RC input |

### Dock operations
| Method | Description |
|--------|--------------|
| `OpenCover(ctx, sn)` | Open the dock cover |
| `CloseCover(ctx, sn, force *bool)` | Close the dock cover (`force` optional) |
| `StartCharging(ctx, sn)` | Start charging |
| `StopCharging(ctx, sn)` | Stop charging |

### Asset management
| Method | Description |
|--------|--------------|
| `RebootAsset(ctx, sn)` | Reboot the asset |
| `BootUpSubAsset(ctx, sn)` | Power on a paired sub-asset (e.g. dock powering on its drone) |
| `BootDownSubAsset(ctx, sn)` | Power off a paired sub-asset |

### Camera and gimbal
| Method | Description |
|--------|--------------|
| `LookAt(ctx, *LookAtRequest)` | Point the gimbal at a coordinate |
| `TakePhoto(ctx, *TakePhotoRequest)` | Capture a photo |
| `ChangeLens(ctx, *ChangeLensRequest)` | Switch camera lens |
| `ChangeZoom(ctx, *ChangeZoomRequest)` | Change zoom level |
| `EnableGimbalTracking(ctx, sn, enabled bool)` | Enable/disable object tracking |

### Live streaming
| Method | Description |
|--------|--------------|
| `StartLiveStream(ctx, *LiveStreamStartRequest)` | Start pushing a live video stream |
| `StopLiveStream(ctx, *LiveStreamStopRequest)` | Stop the live video stream |
| `LiveStreamSplitScreen(ctx, sn, enabled bool)` | Toggle split-screen view across multiple lenses/payloads |

### Debug and maintenance
| Method | Description |
|--------|--------------|
| `EnterRemoteDebugMode(ctx, sn)` | Enter remote debug mode |
| `CloseRemoteDebugMode(ctx, sn)` | Exit remote debug mode |
| `ChangeACMode(ctx, sn, mode string)` | Change the asset-control mode |

### Detection streaming
| Method | Description |
|--------|--------------|
| `GetDetections(ctx, *GetDetectionsRequest, send func(*DetectionResult) error) error` | Server-streaming — call `send` for each detection frame until `ctx` is cancelled or the stream ends; a non-nil error from `send` (or your own return) terminates the stream. |

### Capability reporting
| Method | Description |
|--------|--------------|
| `GetCapabilities(ctx, sn)` | Report the device's current capability snapshot (`*domains.CurrentCapabilities`) |

Each `Capability` carries `Command`, `Description`, `Available`, `UnavailableReason`, `Metadata`,
`TargetType`, `TargetRef`, `SchemaVersion`, `InputSchema`/`OutputSchema` (JSON Schema), `Errors`,
`Events`, `Requirements` and `SkillID` — see `adapter/domains/capability.go`.

### Task execution
| Method | Description |
|--------|--------------|
| `PrepareTask(ctx, sn, taskID, tid string)` | Not called by the 2.0 platform |
| `StartTask(ctx, sn, taskID, tid string)` | Not called by the 2.0 platform |
| `PauseTask(ctx, sn, taskID, tid string)` | Not called by the 2.0 platform |
| `ResumeTask(ctx, sn, taskID, tid string)` | Not called by the 2.0 platform |
| `StopTask(ctx, sn, taskID, tid string)` | **Cancels a running command** — `taskID` is the command's execution id (see below) |

See [Upgrading from 1.3](../concepts/migration-guide.md#task-based-execution-is-gone).

### Custom commands
| Method | Description |
|--------|--------------|
| `SendCustomCommand(ctx, *CustomCommandRequest)` | Handle a command that doesn't map to a standard method above — for example `mission.waypoint.execute`. `CustomCommandRequest` carries `SN`, `TID`, `CommandID`, an optional `TargetRef`, and `Params map[string]any` |

## Reporting a command's outcome

When a command runs as part of a Skill execution, a successful answer means *accepted*, not
finished: the step completes only when your adapter publishes a **command execution event** for it.
That holds for every custom command, and for `TakeOff`, `GoTo`, `LookAt` and `ReturnToHome`. The id
the platform waits on is the request's `TID`, so publish under it — `RUNNING` with `Progress` while it
runs, then exactly one of `CommandExecutionSucceeded` (optionally with `Output`), `CommandExecutionFailed`
or `CommandExecutionCancelled`:

```go
type MyDroneAdapter struct {
    adapter.UnimplementedEdgeAdapter
    live livedata.LiveDataService // set to client.LiveData() after NewEdgeClient
}

func (a *MyDroneAdapter) SendCustomCommand(ctx context.Context, req *domains.CustomCommandRequest) (*domains.CommandResult, error) {
    if req.CommandID != "mission.waypoint.execute" {
        return domains.NotImplemented("unknown command "+req.CommandID, req.SN), nil
    }
    // The request's TID is the id the platform waits on
    go a.flyRoute(req.SN, req.TID, req.Params)
    return domains.SuccessWithTID("route accepted", req.TID, req.SN), nil
}

func (a *MyDroneAdapter) flyRoute(sn, executionID string, params map[string]any) {
    // ... fly the route, publishing CommandExecutionRunning events with Progress as it goes ...
    _ = a.live.PublishCommandExecutionEvent(context.Background(), &domains.CommandExecutionEvent{
        SN:                  sn,
        ExternalExecutionID: executionID,
        CommandID:           "mission.waypoint.execute",
        Status:              domains.CommandExecutionSucceeded,
    })
}
```

Without a terminal event the Skill waits until it times out. To cancel, the platform calls `StopTask`
with that same id.

## `CommandResult`

Every non-streaming command returns `*domains.CommandResult`:

```go
type CommandResult struct {
    Success    bool
    Message    string
    TID        string
    SN         string
    ResultType ResultType // ResultTypeSuccess | ResultTypeError | ResultTypeNotImplemented
}
```

Construct one with the matching helper rather than the struct literal directly:

| Helper | Use |
|--------|-----|
| `domains.Success(message, sn)` | Success, no transaction ID |
| `domains.SuccessWithTID(message, tid, sn)` | Success, echoing the request's transaction ID |
| `domains.Error(message, sn)` | Failure, no transaction ID |
| `domains.ErrorWithTID(message, tid, sn)` | Failure, echoing the request's transaction ID |
| `domains.NotImplemented(message, sn)` | Explicitly not implemented (usually you'd just not override the method and let `UnimplementedEdgeAdapter` handle it instead) |

`result.IsSuccess()` / `result.IsNotImplemented()` are nil-safe convenience checks.
