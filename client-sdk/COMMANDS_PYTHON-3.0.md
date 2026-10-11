# Zequent Client SDK (Python) — Commands (3.0 preview)

> **Preview.** This page describes the Zequent **3.0** command, manual-control and telemetry API of
> the Python Client SDK. 3.0 is
> not released yet: the API below is available from a preview branch and needs a platform that runs
> 3.0. Names may still change before the release. For the current 2.0 line, see
> [Remote Control (Python)](REMOTE_CONTROL_PYTHON.md) and the
> [Python Quickstart](QUICKSTART_PYTHON.md).

On a 3.0 platform every asset is commanded the same way: ask the asset what it can do, then run a
command by its dotted **command id** (`flight.takeoff`, `navigation.go_to`, `dock.open_cover`, …)
with a dict of params. There is no typed method per command, so a command a new adapter adds works
without a new SDK release.

Java: [Commands (Java)](COMMANDS-3.0.md) · Go: [Commands (Go)](COMMANDS_GO-3.0.md) ·
Upgrading an integration: [Upgrading to 3.0](../concepts/upgrading-to-3.0.md).

## Step 1: Add the preview dependency

The SDK is not on PyPI. The 3.0 preview lives on the `feature/v3-agentic` branch of the SDK
and on `main` (the `3.0.0` line) of `zqnt-api` (`zqnt-api-python`), the public Zequent API package it depends on, which replaces
`zqnt-utils` on 3.0 (modules `zqnt_api.…` instead of `zqnt_utils.…`).

`pyproject.toml` (uv-managed projects):

```toml
[project]
dependencies = [
    "zqnt-client-sdk",
]

[tool.uv.sources]
# preview branches, not releases
zqnt-client-sdk = { git = "https://github.com/zequent/zqnt-client-sdk-python", branch = "feature/v3-agentic" }
zqnt-api = { git = "https://github.com/zequent/zqnt-api-python", branch = "main" }
```

then `uv sync` (and `uv lock --upgrade-package zqnt-client-sdk --upgrade-package zqnt-api` to pick
up later preview commits). Move both to the `v3.0.0` release tags once they are out; do not run a
preview in production.

Configuration is unchanged from 2.0: the same environment variables and the same
`ZQNT_CLIENT_TOKEN` client credential (see [Client SDK Configuration (Python)](CONFIGURATION_PYTHON.md)).
`client.commands` uses the remote-control connection (`REMOTE_CONTROL_SERVICE_HOST` / `_PORT`),
credential and interceptors the client already has — nothing else to set up.

## Step 2: Discover what an asset can do

```python
import asyncio

from client_sdk import ZequentClient


async def main() -> None:
    async with ZequentClient.from_env() as client:
        capabilities = await client.commands.list_capabilities("DOCK-1")
        for capability in capabilities.capabilities:
            print(capability.command_id, capability.safety.risk, capability.input_schema)


asyncio.run(main())
```

`await client.commands.list_capabilities(asset_sn)` returns the asset's `CapabilitySet` (the
generated `zqnt_api.generated.zqnt.capability.v3.capability_pb2` message): one `Capability` per
command in `capabilities`, with

- `command_id` — the dotted id you execute, and `display_name` / `description`;
- `input_schema` / `output_schema` — JSON Schemas of the params and of the result;
- `safety.risk` — `COMMAND_RISK_OBSERVE`, `_ADJUST`, `_MOVE` or `_CRITICAL`
  (a command without a rating counts as CRITICAL);
- `state` — whether the command is available right now, with `unavailable_reason`;
- `target` — which part of the asset executes it (the asset, a sub-asset, a payload, a component);
- `completion` — `COMPLETION_MODE_ON_REPLY` (the reply is the outcome) or
  `COMPLETION_MODE_ASYNCHRONOUS` (the outcome follows as an event — takeoff, go-to, return home);
- `errors` / `events` — the error codes and events the command declares.

The set's `snapshot_state` is `SNAPSHOT_STATE_STALE` when the platform could not reach the asset's
adapter and answers with the last snapshot it kept.

The capability's input schema is authoritative for an asset. The built-in commands every Zequent
adapter shares are listed in [Upgrading to 3.0](../concepts/upgrading-to-3.0.md#typed-call--command-id).

## Step 3: Execute a command by id

```python
result = await client.commands.execute_command(
    "DOCK-1", "navigation.go_to", {"latitude": 47.7760, "longitude": 9.2671, "altitude": 60}
)
print(result.state, result.command_execution_id)
```

`await execute_command(asset_sn, command_id, params=None, ...)` returns a `CommandResult`:

- a command that finishes with the reply comes back in its final state (`COMMAND_STATE_SUCCEEDED`
  with `result.result`, or `COMMAND_STATE_FAILED` with `result.error`);
- a command that takes time comes back `COMMAND_STATE_ACCEPTED` or `COMMAND_STATE_RUNNING` with a
  `command_execution_id`; its outcome follows as events (Step 5) — or use `execute_and_wait` (Step 4).

Params are a dict turned into a JSON object. **Leave a param out when you have no value**: the
platform reads a missing coordinate as "not given", while `0` means latitude 0 / longitude 0 /
ground level. A `None` value is left out for you. `navigation.go_to`'s `altitude` is metres above
the **takeoff point** (left out = 40 m).

### Options

All options are keyword-only:

```python
from datetime import timedelta

await client.commands.execute_command(
    "DOCK-1",
    "flight.return_to_home",
    {"altitude": 80},
    timeout=timedelta(minutes=10),
    reason="battery low",
    idempotency_key=my_request_id,
)
```

| Keyword | Meaning |
| --- | --- |
| `asset_id` | Name the asset by its platform id instead of `asset_sn` (pass `None` as `asset_sn`) |
| `target` | A `capability_pb2.Target` — a sub-asset, payload or component instead of the asset itself |
| `timeout` | Seconds or a `timedelta`; replaces the capability's own default timeout |
| `reason` | A human reason; **required** for `COMMAND_RISK_CRITICAL` commands |
| `no_fly_zone_override` | Fly through no-fly zones instead of around them; honoured for an organization admin or a system admin only, anybody else's command is refused |
| `idempotency_key` | The same key sent twice runs the command once. Unset, the SDK uses a fresh key per call, kept across its own retries |

## Step 4: Execute and wait for the outcome

```python
from client_sdk import CommandError
from client_sdk.commands import to_dict

try:
    async with asyncio.timeout(300):
        result = await client.commands.execute_and_wait(
            "DOCK-1", "flight.takeoff", {"latitude": 47.7752, "longitude": 9.2658, "altitude": 40}
        )
except CommandError as error:
    print("not carried out:", error.category_name, error.code, error)
else:
    print("succeeded:", to_dict(result.result))
```

`await execute_and_wait(asset_sn, command_id, params=None, *, target, timeout, reason,
no_fly_zone_override, idempotency_key)` runs the command and returns its outcome:

- It opens a watch on the asset's command events **before** it sends the command, so a run that
  finishes right away is not missed.
- It follows **only the execution id** this call started; events of other runs on the same asset
  are ignored.
- **Only `COMMAND_STATE_SUCCEEDED` is success.** It returns that result; a run that ends
  `REJECTED`, `FAILED`, `CANCELLED` or `TIMED_OUT` raises `CommandError` with the final result in
  `error.result`.
- `timeout=` is the **command's** own timeout, as in `execute_command`. Bound the **wait** with
  `asyncio.timeout(...)`. Leaving the call early (timeout, cancellation) closes the watch and leaves
  **the command running** on the platform; `cancel_command` stops it.
- If the watch ends before the run does, it raises `CommandError` with category
  `ERROR_CATEGORY_SERVICE` (`retryable=True`).
- The asset is named by `asset_sn` (there is no `asset_id` keyword here).

## Step 5: Watch command events

```python
started = await client.commands.execute_command("DOCK-1", "dock.open_cover")
async for event in client.commands.watch_command(started.command_execution_id):
    print(event.state, event.progress, event.message)
```

- `watch_command(command_execution_id)` — an async iterator of one run's events **from now on**,
  ending after its terminal event.
- `watch_asset(asset_sn)` — an async iterator of every command event on the asset from now on,
  until you leave the loop.

Each `CommandEvent` carries `command_execution_id`, `command_id`, `state`, `progress` (0..1, when
the adapter can tell — check `event.HasField("progress")`), `remaining`, `message`, and on the final
event `result` / `error`. A watch the platform refuses or drops raises `CommandError`.

A watch opened after `execute_command` returns can miss the end of a fast run. To wait for a run's
outcome, use `execute_and_wait`.

## Step 6: Cancel a run

```python
cancelled = await client.commands.cancel_command(started.command_execution_id, "operator abort")
```

`await cancel_command(command_execution_id, reason=None)` returns the run's `CommandResult`; a
cancel the platform rejects raises `CommandError`.

## Results and errors

`to_dict(result.result)` (`from client_sdk.commands import to_dict`) turns a result or event payload
into a plain dict. Numbers come back as `float`, as in any JSON object.

Every call raises `CommandError` (`from client_sdk import CommandError`, a `ZequentClientError`)
when:

- the platform refused the call — `error.status` is the `grpc.StatusCode` (`PERMISSION_DENIED`,
  `INVALID_ARGUMENT`, `FAILED_PRECONDITION`, `UNAVAILABLE`, …), including exhausted retries
  (`retryable=True`) and authentication (`ERROR_CATEGORY_PERMISSION_DENIED`);
- the command was `REJECTED` before it started (unknown id, params, precondition, permission) —
  `error.result` is the rejected result, `error.code` its stable code such as `command.invalid_params`;
- from `execute_and_wait` only: the run ended `FAILED`, `CANCELLED` or `TIMED_OUT`.

`error.category` is a `zqnt.common.v3.ErrorCategory` value (`error.category_name` gives its name:
`ERROR_CATEGORY_INVALID_ARGUMENT`, `_PERMISSION_DENIED`, `_PRECONDITION_FAILED`, `_ASSET`,
`_SERVICE`, `_TIMEOUT`, `_NOT_FOUND`); `error.retryable` says whether the same command may succeed
if sent again.

From `execute_command`, a command that **started and then failed** is not an exception: it is
returned with state `COMMAND_STATE_FAILED` and `result.error` (`category`, `code` such as
`flight.not_airborne`, `message`, `retryable`).

## Manual control (stick input)

Stick input goes over one client-streaming session per asset. Put the asset in manual control
first, then send inputs (roll, pitch, yaw, throttle from -1 to 1, optionally a gimbal pitch) at your
control rate:

```python
await client.commands.execute_and_wait("DRONE-1", "flight.manual.enter")
async with client.commands.stream_manual_control("DRONE-1") as stick:
    await stick.send(pitch=0.2, throttle=0.5)
    await stick.send(throttle=0.5, gimbal_pitch=-30.0)
print("inputs passed on:", stick.accepted_inputs)
await client.commands.execute_and_wait("DRONE-1", "flight.manual.exit")
```

- `stream_manual_control(asset_sn)` (not awaited) returns a `ManualControlStream`. One session
  controls one asset; the first input names it. Inputs go out in the order they are sent, without a
  deadline, until the session is closed.
- `await stick.send(roll=0.0, pitch=0.0, yaw=0.0, throttle=0.0, *, gimbal_pitch=None)` sends one input.
- Leaving the `async with` block normally ends the session and waits for the platform's answer;
  `stick.accepted_inputs` is then the number of inputs passed on to the asset. Leaving it with an
  exception aborts the stream.
- Without `async with`: `await stick.close()` ends the session and returns that number;
  `stick.cancel()` aborts the stream without waiting for an answer.
- `send` and `close` raise `CommandError` when the platform refused the session (no adapter
  reachable for the asset, an asset of another organization) or the adapter dropped it; `send`
  after `close` raises `RuntimeError`.

## Live telemetry

`client.telemetry` watches 3.0 telemetry from the live-data service (`LIVE_DATA_SERVICE_HOST` /
`_PORT`, same client credential):

```python
from datetime import timedelta

from zqnt_api.generated.zqnt.telemetry.v3 import telemetry_pb2

async for item in client.telemetry.watch_telemetry(["DOCK-1"], min_interval=timedelta(seconds=1)):
    if isinstance(item, telemetry_pb2.TelemetrySample):
        print(item.asset.sn, item.position, item.battery_percent)
    else:  # SourceStatus
        print(item.asset.sn, item.state)
```

`watch_telemetry(asset_sns=None, *, min_interval=None, heartbeat_timeout=timedelta(seconds=35))` is
an async iterator (not awaited) that yields, until you leave the loop:

- **`TelemetrySample`s** (`zqnt_api.generated.zqnt.telemetry.v3.telemetry_pb2`): the fields every
  asset shares — `position`, `relative_altitude`, `heading_degrees`, `horizontal_speed`,
  `vertical_speed`, `battery_percent`, `observed_at` — and everything device-specific in `details`,
  under the keys the asset declares in its capabilities' `telemetry_fields` (`to_dict(item.details)`
  for a plain dict);
- **`SourceStatus`es**: every watched asset's state first, then each change to
  `SOURCE_STATE_ONLINE`, `SOURCE_STATE_STALE` or `SOURCE_STATE_NO_DATA` (with `last_sample_at`).

No asset (`None` or empty) watches every asset of the caller's organization; a dock's serial also
brings the samples of its drone. `min_interval` (seconds or a `timedelta`) keeps at most one sample
per asset per interval (unset = every sample).

The platform sends heartbeats on a quiet stream: when neither a sample nor a heartbeat arrives within
`heartbeat_timeout` (**35 s by default**, `client_sdk.telemetry.client.DEFAULT_HEARTBEAT_TIMEOUT`), the
loop raises a retryable `CommandError` with category `ERROR_CATEGORY_SERVICE`. A refused or dropped
watch raises `CommandError` too. **A watch does not reconnect**: watch again.

### Detections and alerts

`watch_detections(asset_sns=None, *, heartbeat_timeout=...)` yields the `DetectionBatch`es of the
watched assets (AI detections with the stream they came from); `watch_alerts(asset_sns=None, *,
min_severity=ALERT_SEVERITY_UNSPECIFIED, heartbeat_timeout=...)` yields their `Alert`s (`severity`,
a stable `code` such as `dock.rain`, `message`, `details`) from `min_severity` up. Both are async
iterators with the same heartbeat timeout and errors as `watch_telemetry`, and do not reconnect
either.

## Against a 2.x platform

`client.commands` and `client.telemetry` need a 3.0 platform. A 2.x platform does not serve the 3.0
command and telemetry services: the calls raise `CommandError` with `status` `grpc.StatusCode.UNIMPLEMENTED` (category
`ERROR_CATEGORY_NOT_FOUND`). The 3.0 SDK has no 2.x command or live-data calls: an
application that still has to command a 2.x platform stays on the 2.0 SDK for it. See
[Upgrading to 3.0](../concepts/upgrading-to-3.0.md#running-against-2x-platforms).
