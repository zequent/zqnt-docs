# Upgrading to 3.0 (preview)

> **Preview.** Zequent **3.0** is not released yet. **2.0.x** stays the current long-term-support
> line and its documentation is unchanged. This page describes what changes for a **Client SDK**
> integration when you move it to the 3.0 SDKs, which are available as preview builds. Names may
> still change before the release.

## What changes

In 2.x, the Client SDKs had one typed method per command (`takeoff`, `goTo`, `openCover`, …). In
3.0 every action on an asset is a **capability** with a dotted **command id** (`flight.takeoff`,
`navigation.go_to`, `dock.open_cover`, `vendor.<vendor>.<name>`) and JSON Schemas for its params and
result. Every SDK has one way to run any of them:

| | Java | Python | Go |
| --- | --- | --- | --- |
| Entry point | `zequent.commands()` | `client.commands` | `control.New(conn)` |
| What can this asset do? | `listCapabilities(assetSn)` | `list_capabilities(asset_sn)` | `ListCapabilities(ctx, sn, ...)` |
| Run a command | `executeCommand(assetSn, commandId, params)` / `executeCommand(CommandRequest)` | `execute_command(asset_sn, command_id, params, ...)` | `ExecuteCommand(ctx, sn, commandID, params, ...)` |
| Run and wait for the outcome | `executeAndWait(assetSn, commandId, params, wait)` | `execute_and_wait(asset_sn, command_id, params, ...)` | `ExecuteAndWait(ctx, sn, commandID, params, ...)` |
| Follow a run / an asset | `watchCommand(id, onEvent)` / `watchAsset(assetSn, onEvent)` | `watch_command(id)` / `watch_asset(asset_sn)` | `WatchCommand(ctx, id)` / `WatchAsset(ctx, sn)` |
| Cancel a run | `cancelCommand(id, reason)` | `cancel_command(id, reason)` | `CancelCommand(ctx, id, reason)` |
| Stick input | `streamManualControl(assetSn)` | `stream_manual_control(asset_sn)` | `StreamManualControl(ctx, sn)` |
| Live telemetry | `zequent.telemetry().watchTelemetry(assetSns, minInterval, onSample, onSourceStatus)` | `client.telemetry.watch_telemetry(asset_sns, min_interval=...)` | `telemetry.New(conn).WatchTelemetry(ctx, sns, ...)` |
| Guide | [Commands (Java)](../client-sdk/COMMANDS-3.0.md) | [Commands (Python)](../client-sdk/COMMANDS_PYTHON-3.0.md) | [Commands (Go)](../client-sdk/COMMANDS_GO-3.0.md) |

A command a new adapter adds is usable the moment the adapter publishes it — no new SDK release.

### Manual control

Stick input goes over one client-streaming session per asset (`StreamManualControl`): enter manual
control with the command `flight.manual.enter`, open the session, send roll, pitch, yaw and throttle
(-1..1, optionally a gimbal pitch) at your control rate, close it — the close answers with the number
of inputs passed on to the asset — and leave with `flight.manual.exit`.

| 2.x | 3.0 |
| --- | --- |
| Java `remoteControl().startManualControlInput(sn, assetId)` (deprecated in the 3.0 SDK) | `commands().streamManualControl(assetSn)` → `ManualControlStream`: `send(...)`, `close()`, `done()` |
| Python `remote_control.start_manual_control_input(sn)` (deprecated in the 3.0 SDK) | `commands.stream_manual_control(asset_sn)` → `ManualControlStream`: `async with`, `send(...)`, `close()`, `cancel()` |
| Go `remotecontrol.Client.OpenManualControlInputStream(ctx)` (package deprecated) | `control.Client.StreamManualControl(ctx, sn)` → `Send(...)`, `Close()`; `Send` after `Close` is `control.ErrManualControlClosed` |

### Live telemetry

3.0 telemetry is one stream of **samples** and **source status** from live-data. A
`TelemetrySample` has the fields every asset shares (position, relative altitude, heading, speeds,
battery); everything device-specific is in its `details`, under the keys the asset declares in its
capabilities' `telemetry_fields`. A `SourceStatus` reports each watched asset as online, stale or
without data — first for every watched asset, then on each change. No asset watches every asset of
the caller's organization; a dock's serial also brings its drone's samples. A minimum interval keeps
at most one sample per asset per interval.

The platform sends heartbeats on a quiet stream; every SDK ends the watch with an error when neither
a sample nor a heartbeat arrives within **35 s** by default (configurable). **No SDK reconnects a
telemetry watch on its own** — watch again.

| 2.x | 3.0 |
| --- | --- |
| Java `liveData().streamTelemetryData(...)` | `telemetry().watchTelemetry(assetSns, minInterval, onSample, onSourceStatus)` → `TelemetryWatch` |
| Python `live_data.stream_telemetry(...)` | `async for item in telemetry.watch_telemetry(asset_sns, min_interval=...)` |
| Go `livedata.Client.StreamTelemetry(ctx, sn, frequencyMs, durationSeconds)` | `telemetry.New(conn).WatchTelemetry(ctx, sns, telemetry.WithMinInterval(...))` → `Recv()` |

Every command run has a **command execution id** and a state: `ACCEPTED` / `RUNNING` while underway,
then one of `SUCCEEDED`, `FAILED`, `REJECTED` (refused before it started), `CANCELLED` or
`TIMED_OUT`. A command whose reply is its outcome (open a cover) comes back finished; one that takes
time (takeoff, go-to, return home) comes back `ACCEPTED` and its outcome follows as a command event.

### Waiting for the outcome: `executeAndWait`

All three SDKs wait the same way:

1. They open a watch on the asset's command events **before** they send the command, so a run that
   finishes right away is not missed.
2. They follow **only the execution id** that call started; events of other runs on the asset are
   ignored.
3. **Only `SUCCEEDED` is success.** `REJECTED`, `FAILED`, `CANCELLED` and `TIMED_OUT` all end in the
   SDK's command error, carrying the final result.
4. Giving up the wait (the Java `wait` duration, `asyncio.timeout` in Python, the `ctx` deadline in
   Go) does **not** stop the command on the platform. Cancel it explicitly.

Watching a run yourself after `executeCommand` has returned can miss the end of a fast run; wait
with `executeAndWait` instead.

### What stays the same

- Configuration, environment variables and the `ZQNT_CLIENT_TOKEN` client credential. The 3.0
  command service is served by remote-control next to the 2.x service, on the same host and port.
- Connector and Mission Autonomy (Applications, Skill executions, schedulers) keep their 2.x
  contracts in the 3.0 SDKs. The 2.x Live Data client stays next to the new telemetry watch.
- `navigation.go_to`'s `altitude` is metres above the **takeoff point** (left out = 40 m), as in 2.x.
- **Leave a param out when you have no value.** The platform reads a missing coordinate as "not
  given"; sending `0` means latitude 0 / longitude 0 / ground level. Java and Python drop `null` /
  `None` values for you.

## Installing the 3.0 SDK previews

| SDK | Preview |
| --- | --- |
| Java | `com.zqnt.sdk:client-java-sdk` preview builds on GitHub Packages, `3.0.0-PR<n>-<sha>-SNAPSHOT` (at the time of writing `3.0.0-PR12-869caa4-SNAPSHOT`); it pulls `zqnt-api-java` 3.0.0. See [Commands (Java)](../client-sdk/COMMANDS-3.0.md#step-1-add-the-preview-dependency). |
| Python | Git branch `feature/v3-agentic` of `zqnt-client-sdk-python` **and** of `zqnt-utils-python`, under `[tool.uv.sources]`. See [Commands (Python)](../client-sdk/COMMANDS_PYTHON-3.0.md#step-1-add-the-preview-dependency). |
| Go | New module path **`github.com/zequent/zqnt-client-sdk-go/v3`** (`/v3`, lower-case owner): `go get github.com/zequent/zqnt-client-sdk-go/v3@feature/v3-agentic`. Its generated types come from the private module `github.com/zequent/zqnt-utils-golang/v2`. See [Commands (Go)](../client-sdk/COMMANDS_GO-3.0.md#step-1-add-the-preview-module). |

Preview builds are for trying 3.0 against a 3.0 platform, not for production. Each moves to a
`3.0.0` release once 3.0 ships.

## Running against 2.x platforms

The 3.0 command, manual-control and telemetry APIs need a platform that runs 3.0. A 2.x platform
does not serve them, and every call fails with the gRPC status `UNIMPLEMENTED`:

| SDK | How it shows |
| --- | --- |
| Java | the future fails with `CommandException`, `getStatus()` = `UNIMPLEMENTED`, category `ERROR_CATEGORY_NOT_FOUND` |
| Python | `CommandError`, `status` = `grpc.StatusCode.UNIMPLEMENTED`, category `ERROR_CATEGORY_NOT_FOUND` |
| Go | `status.Code(err) == codes.Unimplemented` |

The Client SDKs **do not switch to the 2.x calls on their own**. The 2.x typed calls are still in
the 3.0 SDKs and keep working against 2.x platforms — deprecated, not removed:

- Java: the typed `remoteControl()` methods, `startManualControlInput` and `getCapabilities` /
  `sendCustomCommand` are `@Deprecated(since = "3.0")`.
- Python: the typed `client.remote_control` methods and `start_manual_control_input` emit a
  `DeprecationWarning` that names what to use on 3.0.
- Go: package `remotecontrol` is deprecated and stays in the `/v3` module.

So an application that has to talk to both 2.x and 3.0 platforms decides per platform which API to
use — for example from configuration, or by treating `UNIMPLEMENTED` from `listCapabilities` as
"this platform is still on 2.x":

```java
boolean platformServes3(ZequentClient zequent, String assetSn) {
    try {
        zequent.commands().listCapabilities(assetSn).join();
        return true;
    } catch (CompletionException failure) {
        CommandException error = CommandException.from(failure);
        if (error.getStatus() == io.grpc.Status.Code.UNIMPLEMENTED) return false; // 2.x: use remoteControl()
        throw error;
    }
}
```

Moving the platform itself to 3.0 is a separate upgrade, documented with the 3.0 release.

## Typed call → command id

`listCapabilities` on an asset is authoritative for the commands and params it accepts. These are
the ids and params of the built-in commands the Zequent adapters share:

| 2.x Java (`remoteControl()`) | 2.x Python (`remote_control`) | 2.x Go (`remotecontrol`) | Command id | Params |
| --- | --- | --- | --- | --- |
| `takeoff` | `takeoff` | `TakeOff` | `flight.takeoff` | `latitude`, `longitude`, `altitude` |
| `goTo` | `go_to` | `GoTo` | `navigation.go_to` | `latitude`, `longitude`, `altitude` (above the takeoff point; left out = 40 m) |
| `goTo(request, true)` | `go_to(request, no_fly_zone_override=True)` | `GoToWithOptions(..., GoToOptions{NoFlyZoneOverride: true})` | `navigation.go_to` | as above, plus the no-fly zone override option of the 3.0 call |
| `returnToHome` | `return_to_home` | `ReturnToHome` | `flight.return_to_home` | `altitude` (optional) |
| `lookAt` | `look_at` | `LookAt` | `gimbal.look_at` | `latitude`, `longitude`, `altitude`, `locked`, `payloadIndex` |
| `enterManualControl` / `exitManualControl` | `enter_manual_control` / `exit_manual_control` | `EnterManualControl` / `ExitManualControl` | `flight.manual.enter` / `flight.manual.exit` | none |
| `openCover` | `open_cover` | `OpenCover` | `dock.open_cover` | none |
| `closeCover` | `close_cover` | `CloseCover` | `dock.close_cover` | `force` (boolean, optional) |
| `startCharging` / `stopCharging` | `start_charging` / `stop_charging` | `StartCharging` / `StopCharging` | `dock.start_charging` / `dock.stop_charging` | none |
| `rebootAsset` | `reboot_asset` | `RebootAsset` | `asset.reboot` | none |
| `bootSubAsset` | `boot_sub_asset` | `BootSubAsset` | `asset.boot_sub_asset` | `enabled` (boolean: `true` boots, `false` shuts down) |
| `debugMode` | `debug_mode` | `SetRemoteDebugMode` | `asset.remote_debug` | `enabled` (boolean) |
| `changeAcMode` | `change_ac_mode` | `ChangeAcMode` | `asset.change_ac_mode` | `mode` (one of the values in the capability's schema) |
| `takePhoto` | — | `CapturePhoto` | `camera.take_photo` | none |
| `liveStreamSplitScreen` | — | `LiveStreamSplitScreen` | `stream.split_screen` | `enabled` (boolean) |
| `liveData().changeCameraLens` | `live_data.change_lens` | `ChangeLens` | `camera.change_lens` | `lens`, `videoId` |
| `liveData().changeCameraZoom` | `live_data.change_zoom` | `ChangeZoom` | `camera.change_zoom` | `lens`, `payloadIndex`, `zoom` |
| `getCapabilities(sn)` | — | `GetCapabilities` | — | use `listCapabilities` / `list_capabilities` / `ListCapabilities` |
| `sendCustomCommand(...)` | — | `SendCustomCommand` | the capability's command id | the same params; the 2.x capability target becomes the 3.0 call's target option |

## Responses and errors

| 2.x | 3.0 |
| --- | --- |
| `success` / `has_errors` on the response | The result's state: `SUCCEEDED`, `FAILED`, … — or, with `executeAndWait`, the `SUCCEEDED` result or the SDK's command error |
| error code and message on the response | The result's `error`: `category`, a stable `code` (e.g. `flight.not_airborne`, `command.invalid_params`), `message`, `retryable` |
| a refused call (gRPC status) | Java `CommandException` / Python `CommandError` with the gRPC status; Go keeps the gRPC status on the error |
| `progress` on the response | Command events: `progress`, `remaining`, `message`, and the final `result` |
| `tid` | The command execution id: watch it, cancel it |
| a custom command's result map | The result payload: Java `Structs.toMap(...)`, Python `to_dict(...)`, Go `GetResult().AsMap()` |

One difference between the SDKs: from a plain `executeCommand`, **Java and Python** return a run that
started and failed as a result in state `FAILED`, while **Go** returns it as a `*control.CommandError`.
All three raise their command error for a `REJECTED` command, and all three treat only `SUCCEEDED`
as success in `executeAndWait`.

New on 3.0, per call: a **reason** (required for commands rated `COMMAND_RISK_CRITICAL`), an
**idempotency key** (the same key sent twice runs the command once), a **timeout** that replaces the
capability's default, and a **target** for a sub-asset, payload or component.

## Go: module path

Go callers change their import paths once — the client module moves to `/v3` (lower-case owner) and
the generated types move to `zqnt-utils-golang`, with the same package paths below `gen/`:

```bash
grep -rl zqnt-client-sdk-go/v2 --include='*.go' . | xargs sed -i \
  -e 's#github.com/Zequent/zqnt-client-sdk-go/v2/gen/#github.com/zequent/zqnt-utils-golang/v2/gen/#g' \
  -e 's#github.com/Zequent/zqnt-client-sdk-go/v2/#github.com/zequent/zqnt-client-sdk-go/v3/#g'
go get github.com/zequent/zqnt-client-sdk-go/v3@feature/v3-agentic && go mod tidy
```
