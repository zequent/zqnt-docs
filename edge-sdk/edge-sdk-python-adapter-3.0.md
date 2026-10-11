# Zequent Edge SDK (Python) — Adapters on 3.0 (preview)

> **Preview.** This page describes the **3.0** adapter API of the Python Edge SDK. 3.0 is not
> released yet: the API below is available as a preview. Names may still change before the
> release. For the current 2.0 line, see [Edge Adapter (Python)](edge-sdk-python-adapter.md) and the
> [Python Quickstart](edge-sdk-python-quickstart.md).

On 3.0 an adapter declares each command **once** with `register_command`: a dotted command id, a
handler and (optionally) JSON Schemas for params and result. What the asset advertises, the v3
`ExecuteCommand` the platform calls and the v2 typed RPCs a 2.x platform still calls all come from
that registration, so the same adapter works against 2.x and 3.0 platforms.

Java: [Adapters on 3.0 (Java)](edge-sdk-adapter-3.0.md) · Go: [Adapters on 3.0 (Go)](edge-sdk-go-adapter-3.0.md) ·
Platform: [Upgrading the platform to 3.0](../concepts/upgrading-platform-to-3.0.md).

## Step 1: Add the preview dependency

The SDK (`edge-python-sdk`, version line `3.0.0`) is not on PyPI. Its generated types and helpers
now come from **`zqnt-api`** (`zqnt-api-python`, public, Apache 2.0), which replaces `zqnt-utils`;
modules move from `zqnt_utils.…` to `zqnt_api.…` (for example
`zqnt_api.generated.zqnt.telemetry.v3`). The SDK preview lives on the `feature/public-api-split`
branch, `zqnt-api` on `main` (its `3.0.0` line):

```toml
[project]
dependencies = [
    "edge-python-sdk",
]

[tool.uv.sources]
# preview branches, not releases
edge-python-sdk = { git = "https://github.com/zequent/zqnt-edge-sdk-python", branch = "feature/public-api-split" }
zqnt-api = { git = "https://github.com/zequent/zqnt-api-python", branch = "main" }
```

uv allows one source per package in a dependency graph, so pin `zqnt-api` to the same ref the SDK
uses. Run `uv lock --upgrade-package edge-python-sdk --upgrade-package zqnt-api` to pick up later
preview commits, and move both to the `v3.0.0` tags once they are out; do not run a preview in
production. Configuration and authentication (`ZQNT_EDGE_TOKEN`, `ZQNT_PLATFORM_PUBLIC_KEY`,
`ZQNT_EDGE_AUTH_DISABLED`) are unchanged from 2.0 — see
[Edge SDK Configuration (Python)](edge-sdk-python-configuration.md).

## Step 2: Declare the commands

```python
from edge_sdk import (
    AssetType,
    CustomCommandResponse,
    EdgeAdapter,
    ErrorCode,
    ErrorMessage,
    TelemetryValueType,
    schema,
)


class AcmeDock(EdgeAdapter):
    def __init__(self, link):
        self._link = link
        self.register_command("dock.open_cover", self._open_cover)
        self.register_command(
            "vendor.acme.spray",
            self._spray,
            description="Run the spray boom for N seconds",
            input_schema=schema({"seconds": {"type": "integer", "minimum": 1}}, ["seconds"]),
            skill_id="acme",
        )
        self.declare_standard_telemetry_field("dock.cover_state")
        self.declare_telemetry_field("vendor.acme.tank_level", TelemetryValueType.NUMBER, unit="%")

    async def get_capabilities(self, sn, asset_id):
        return self._auto_capabilities(sn, AssetType.DOCK)

    def reported_asset_sns(self):
        return ["DOCK-1"]

    async def _open_cover(self, ctx, params):
        await self._link.open_cover()
        return CustomCommandResponse.ok(ctx.tid, ctx.sn, "dock.open_cover")

    async def _spray(self, ctx, params):
        try:
            job = await self._link.spray(params["seconds"])
        except OSError as e:
            return CustomCommandResponse.fail(
                ctx.tid, ctx.sn, "vendor.acme.spray", ErrorMessage(str(e), ErrorCode.ASSET_ERROR)
            )
        return CustomCommandResponse.ok(ctx.tid, ctx.sn, "vendor.acme.spray", external_execution_id=job.id)
```

- **Ids.** Reuse the platform's ids for shared actions (`flight.takeoff`, `navigation.go_to`,
  `dock.open_cover`, …); for those, `register_command` takes description, schemas, display name and
  `skill_id` from the platform catalog the SDK ships (`edge_sdk.COMMAND_CATALOG`), so registering
  them is one line. A device-specific command goes under `vendor.<vendor>.<name>` and brings its own
  schema; pass `skill_id` so the console groups it with its siblings.
- **Handler.** `async def handler(ctx, params) -> CustomCommandResponse`. `ctx.sn` is the asset,
  `ctx.command_execution_id` the platform's id of this run (v3; `None` over v2).
- **Validation.** Params are validated against the input schema before the handler runs. Integral
  numbers become `int` where the schema says `integer`; invalid params come back `REJECTED` with
  `command.invalid_params`. Omitted coordinates arrive as NaN: NaN is a number, but never satisfies
  `required`.
- **Result.** `CustomCommandResponse.ok(...)` finishes the command (`result=` is its output).
  `ok(..., external_execution_id=...)` means "still running" — v3 `ACCEPTED` — and the outcome follows
  as a command event (Step 4). `CustomCommandResponse.fail(...)` with an `ErrorMessage` fails it.
- **Changing what the asset can do.** Registering an id again replaces it (for example with
  `state=CapabilityState.TEMPORARILY_UNAVAILABLE` while a payload is detached);
  `unregister_command(id)` withdraws it. Both report the change to the platform;
  `notify_capabilities_changed(sn)` does so for anything else.
- `get_capabilities` stays the one required method; `_auto_capabilities` builds the snapshot from the
  registrations and the catalog (catalog ids the adapter does not run are reported `UNSUPPORTED`).

The typed methods (`take_off`, `open_cover`, …) are the v2 compatibility layer: overriding one
still works and is served under its dotted id, but warns with a `DeprecationWarning`.

### Command risk (`CommandSafety`)

The 3.0 contract rates every command with a `CommandSafety` (`COMMAND_RISK_OBSERVE`, `_ADJUST`,
`_MOVE`, `_CRITICAL`, plus reversibility and an undo command). A standard id is advertised with
the command catalog's rating (`edge_sdk.adapter.commands.catalog_safety(id)`: `flight.takeoff` MOVE,
`flight.land` CRITICAL, ...). Declare your own at registration:

```python
from edge_sdk import CommandRisk, CommandSafety

self.register_command("vendor.acme.spray", self._spray, safety=CommandRisk.CRITICAL)
self.register_command(
    "dock.open_cover", self._open_cover,
    safety=CommandSafety(risk=CommandRisk.MOVE, preconditions=("no aircraft above the dock",)),
)
```

A `vendor.*` command that moves or affects the asset must declare its own risk. An id outside the catalog
that declares none counts as **critical** (a client must give a reason to run it).

### Payloads

A device mounted on the asset (camera, gimbal, speaker, spotlight, sprayer, sensor) is reported with
the capabilities. Set the asset's payloads on the adapter, and target a payload's commands at its
`ref`:

```python
from edge_sdk import Payload
from edge_sdk.models.common import CapabilityTarget, CapabilityTargetType

self.set_payloads("SN-1", [Payload(ref="speaker-1", kind="speaker", vendor="Acme", model="S2", slot_index=0)])
self.register_command(
    "speaker.audio.play",
    self._play,
    target=CapabilityTarget(type=CapabilityTargetType.PAYLOAD, target_ref="speaker-1"),
)
```

`set_payloads` reports again when the snapshot changed; `mounted_on` names the sub-asset carrying
the payload (`None`: the asset itself). A report is a full snapshot: a payload missing from the next report is detached. The platform keeps
the payloads on the asset (the console's payload list) and stores one mounted on a sub-asset (a
drone of a dock) on that sub-asset. `kind` is free text (`camera`, `speaker`, `spotlight`,
`sprayer`, `sensor`, ...); `metadata` carries anything else, `library_version` is stored as such.
The conformance kit
flags a payload without `ref`, two with the same one, and a command targeting a payload that is not
reported.

## Step 3: Serve and connect

`EdgeServer` serves `zqnt.edge.v3.EdgeAdapterService` (`ExecuteCommand`) and the v2 service side by
side from the same registrations. The simplest wiring is the runtime, configured from the
environment:

```python
import asyncio

from edge_sdk import EdgeAdapterConfig


async def main():
    async with EdgeAdapterConfig.from_env().runtime() as rt:
        await rt.serve(AcmeDock(link))


asyncio.run(main())
```

Set `REMOTE_CONTROL_HOST` (and `REMOTE_CONTROL_PORT`, default `8002`) so the runtime creates an
`EdgeGatewayClient` (`rt.gateway`): capability reports and v3 command events go through it.
Without it there is no capability push and command events go over v2. With your own `EdgeServer`,
pass `gateway=EdgeGatewayClient(host, port)`.

| What | 3.0 platform | 2.x platform (answers `UNIMPLEMENTED`) |
| --- | --- | --- |
| Commands from the platform | v3 `ExecuteCommand` | v2 RPCs, served alongside |
| Capabilities | `ReportCapabilities` (with `telemetry_fields`) on start and on every change, for `reported_asset_sns()` and every serial passed to `notify_capabilities_changed` | v2 `ReportAssetRuntime` |
| Command events | `PublishCommandEvent` under the platform's `command_execution_id` | v2 notification stream |
| Telemetry, detections | `TelemetryIngestPublisher` (`rt.ingest`), v3 `TelemetryIngestService` | v2 `ProduceTelemetry` / `ProduceDetection`, catalog keys mapped — see Step 5 |
| Alerts | `TelemetryIngestPublisher.publish_alert` | dropped (no v2 counterpart) |

**Automatic fallback.** After `UNIMPLEMENTED` the SDK uses v2 for **10 minutes**
(`V3Fallback.WINDOW_SECONDS = 600`), per service and process, then tries v3 again — an adapter picks
up a platform upgrade without a restart.

## Step 4: Report command events

A command that answered with an `external_execution_id` reports its progress and outcome as
before, through `NotificationPublisher`:

```python
from edge_sdk import CommandExecutionEvent, CommandExecutionStatus

await notifier.publish_command_execution_event(
    CommandExecutionEvent(
        external_execution_id=job.id,
        status=CommandExecutionStatus.SUCCEEDED,
        sn="DOCK-1",
        command_id="vendor.acme.spray",
        output={"litres": 12.5},
    )
)
```

With a gateway (`NotificationPublisher(..., gateway=...)`, or `REMOTE_CONTROL_HOST` set) the event
goes to `EdgeGatewayService.PublishCommandEvent` under the platform's `command_execution_id` —
looked up from the `external_execution_id` the command was accepted under, or set
`command_execution_id` yourself. `occurred_at` is always set (now, if omitted). Progress belongs on a
`RUNNING` event (`progress` 0..1); finish every accepted command with exactly one of `SUCCEEDED`,
`FAILED`, `CANCELLED`.

## Step 5: Publish telemetry

```python
from edge_sdk import Alert, AlertSeverity, TelemetrySample

await rt.ingest.publish_sample(
    TelemetrySample(
        sn="DOCK-1",
        latitude=47.5,
        longitude=9.7,
        battery_percent=81.0,
        details={"dock.cover_state": "OPEN", "vendor.acme.tank_level": 63.0},
    )
)
await rt.ingest.publish_alert(Alert(sn="DOCK-1", code="dock.rain", severity=AlertSeverity.WARNING))
```

`TelemetrySample` has the fields every asset shares — `latitude`, `longitude`, `altitude` (above sea
level), `relative_altitude` (above the takeoff point), `heading_degrees`, `horizontal_speed`,
`vertical_speed`, `battery_percent` — and `details` for everything device-specific. `None` and NaN
values are not sent; `observed_at` defaults to now. `TelemetryIngestPublisher` keeps one long-lived
stream per kind (telemetry, detections, alerts), buffers while it is down and reconnects with
backoff (1 s doubling to 60 s). `publish_detections(DetectionBatch(...))` publishes detections.

**Standard keys.** The platform's telemetry catalog (`zqnt_api.telemetry`) has 64 detail keys —
listed in [The telemetry catalog](edge-sdk-adapter-3.0.md#the-telemetry-catalog) — with constants
(`zqnt_api.telemetry.DOCK_COVER_STATE`, …), `telemetry_fields()` and `telemetry_field(key)`.
`declare_standard_telemetry_field(key)` declares one with the platform's type, unit and allowed
values (and raises `ValueError` for a key outside the catalog).

**On a 2.x platform** samples and detections go over v2 `ProduceTelemetry` / `ProduceDetection`,
mapped by `zqnt_api.telemetry`: **catalog keys in `details` land in their v2 fields, other keys are
dropped**. A sample with a speed or an aircraft key becomes aircraft (sub-asset) telemetry, one with
dock keys or only a battery asset telemetry.

The v2 `TelemetryPublisher` (`publish_asset_telemetry`, `publish_subasset_telemetry`) is
unchanged and still works against both platforms.

## Pairing with a claim code

Pairing is unchanged from 2.0: with `ZQNT_CLAIM_CODE` set (a one-time code minted in the console),
`rt.connector.ensure_asset(asset)` binds to an asset the platform already knows and redeems the code
only for an unknown serial; `redeem_asset_claim(code, asset)` is there for custom flows. See
[Edge Connector (Python) — pairing](edge-sdk-python-connector.md). The Python SDK has no
`register_asset` on the connector; the platform's organization-UUID bind is **planned** to be
removed in 3.0 — pair with claim codes.

## Step 6: Check the adapter with the conformance kit

```python
from edge_sdk.testing import RecordingGateway, assert_conformant


async def test_adapter_is_conformant():
    await assert_conformant(AcmeDock(fake_link), sn="SIM-1")
    # also runs every advertised command with minimal params -- fakes or simulators only
    await assert_conformant(AcmeDock(fake_link), sn="SIM-1", execute=True)


async def test_completion_events():
    gateway = RecordingGateway()
    ...  # NotificationPublisher(..., gateway=gateway), run a command
    gateway.assert_events_complete()
```

`assert_conformant` checks that every advertised id is executable, every executable id is
advertised and the schemas parse (`check_conformance` returns the `ConformanceReport` instead of
raising). `RecordingGateway` records command events and capability reports;
`assert_events_complete()` checks that every recorded event carries `occurred_at`, the
`command_execution_id` and the asset's serial.
