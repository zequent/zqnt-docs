# Edge SDK (Python) — Live Data

`LiveDataService` is the recommended facade for all outbound data from an adapter — it composes a `TelemetryPublisher`, a `DetectionPublisher`, and notification publishing behind one connection lifecycle. (`TelemetryPublisher` and `DetectionPublisher` are also available standalone if you only need one of the three.)

Full method-by-method reference, including reconnection/queueing behavior: [Live Data API Reference](../api-reference/edge-sdk-python-live-data-reference.md).

For Java, see [edge-sdk-live-data.md](edge-sdk-live-data.md).

---

## Lifecycle

```python
from edge_sdk import LiveDataService

async with LiveDataService(host="localhost", port=8003, sn="DOCK-1") as live:
    await live.produce_telemetry(asset_telemetry)
```

Or manage it explicitly:

```python
live = LiveDataService(host="localhost", port=8003, sn="DOCK-1")
await live.connect()
try:
    await live.produce_telemetry(asset_telemetry)
finally:
    await live.close()
```

---

## Telemetry

`AssetTelemetry` and `SubAssetTelemetry` are flat dataclasses — position (`latitude`/`longitude`/`absolute_altitude`/`relative_altitude`) and movement fields live directly on them, not nested in a separate position object. `AssetPositionState`/`SubAssetBatteryInfo`/etc. are for the specific sub-structures that really are nested (GNSS fix quality, battery detail).

### Asset telemetry

```python
from datetime import datetime, timezone
from edge_sdk import AssetTelemetry, AssetMode

asset = AssetTelemetry(
    id="DOCK-1",
    timestamp=datetime.now(tz=timezone.utc),
    latitude=47.3769,
    longitude=8.5417,
    absolute_altitude=450.0,
    mode=AssetMode.WORKING,
    environment_temp=22.5,
    humidity=65.0,
)
await live.produce_telemetry(asset)
```

All fields except `id` are optional — populate only what your device measures.

### Sub-asset telemetry

```python
from edge_sdk import SubAssetTelemetry, SubAssetMode, SubAssetBatteryInfo

sub = SubAssetTelemetry(
    id="DRONE-1",
    timestamp=datetime.now(tz=timezone.utc),
    latitude=47.3769,
    longitude=8.5417,
    absolute_altitude=120.0,
    horizontal_speed=5.2,
    mode=SubAssetMode.MANUAL,
    battery=SubAssetBatteryInfo(percentage=72),
)
await live.produce_telemetry(sub)
```

`produce_telemetry` accepts either `AssetTelemetry` or `SubAssetTelemetry` — the SDK routes it correctly based on the type you pass.

### Which one should you publish?

The choice is about **which entity the reading describes**, not what kind of device it is:

- Publish **`AssetTelemetry`** when the reading describes the registered top-level **Asset** itself. An Asset can be a drone, a dock, a ground vehicle, a sensor gateway, a camera or a station. If your adapter registers a drone as a standalone Asset with no parent, its telemetry is `AssetTelemetry` — that is correct, and the platform records it with `source_type = ASSET` and no sub-asset.
- Publish **`SubAssetTelemetry`** when the reading describes an optional child **SubAsset** belonging to an Asset — for example the drone that lives in a dock, where the dock is the Asset.

`AssetTelemetry` is a superset covering every kind of Asset, so a standalone drone simply leaves the dock-only fields (`cover_state`, `air_conditioner`, `inside_temp`, `working_voltage`) as `None`. That is complete telemetry, not partial.

See [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) for fully-populated examples of both configurations, and [Models Reference](../api-reference/edge-sdk-python-models.md#telemetry) for the complete field list.

---

## Detections

```python
from edge_sdk import DetectionBatch, DetectionResult, BoundingBox

await live.produce_detection(
    DetectionBatch(
        sn="DRONE-1",
        stream_url="rtmp://...",
        detections=[
            DetectionResult(
                object_id="obj-1",
                object_type="person",
                confidence=0.91,
                bounding_box=BoundingBox(x=120, y=80, width=64, height=128),
            ),
        ],
    )
)
```

---

## Notifications

Three cases: reporting an asset's online/offline transitions, reporting progress or completion of a task your adapter is running, and (less commonly — no confirmed usage in any current adapter) mission-level events. See the [reference](../api-reference/edge-sdk-python-live-data-reference.md#notifications) for `MissionEvent`'s fields.

```python
from edge_sdk import AssetStatusEvent, TaskEvent, TaskStatus, TaskType

# Asset went offline
await live.produce_notification(
    AssetStatusEvent(sn="DOCK-1", online=False, message="Lost connection to device")
)

# Progress for a task your adapter is running
await live.produce_notification(
    TaskEvent(
        task_id=task_id,
        task_type=TaskType.WAYPOINT,
        status=TaskStatus.RUNNING,
        sn="DRONE-1",
        progress=0.42,
    )
)
```

---

## Error handling and reconnection

`produce_*` calls don't raise for a disconnected stream — each one enqueues onto an internal bounded queue (1000 items by default) and returns immediately; the background stream task retries the actual gRPC call with exponential backoff (1s up to 60s, no attempt limit) and drains the queue once reconnected. If you publish faster than the queue drains while disconnected, new calls made once the queue is full are silently dropped (logged at `debug`, not raised) — see the [reference](../api-reference/edge-sdk-python-live-data-reference.md#reconnection-and-queueing-behavior) for the exact behavior. A simple producer loop needs no explicit error handling:

```python
while True:
    await live.produce_telemetry(read_from_device())
    await asyncio.sleep(0.1)
```

The underlying stream reconnects automatically when the server drops it — you don't need to call `connect()` again.

---

## Performance tips

- **Push at a fixed cadence**, not on every sensor reading. 1–10 Hz is typical for asset telemetry; 0.5–2 Hz for sub-asset.
- **Reuse a single `LiveDataService`** for an entire process. Don't create one per push.
- **Use `asyncio.gather`** if you need to push asset and sub-asset telemetry concurrently from independent tasks.

---

## Sending telemetry from inside an adapter method

```python
class MyAdapter(EdgeAdapter):
    def __init__(self, live: LiveDataService):
        self._live = live

    async def take_off(self, ctx, coordinates):
        result = await drone.takeoff(...)
        await self._live.produce_telemetry(result.to_asset_telemetry())
        return EdgeResponse.ok(ctx.tid, ctx.sn, "Takeoff initiated")
```

Wire it in your `main`. If you're using `EdgeAdapterConfig`/`EdgeAdapterRuntime` (see [Quickstart](edge-sdk-python-quickstart.md)), `runtime.telemetry` already gives you a connected `TelemetryPublisher` for the telemetry-only case; construct a standalone `LiveDataService` alongside it if you also need detections/notifications:

```python
config = EdgeAdapterConfig.from_env()
async with config.runtime() as runtime:
    live = LiveDataService(host=config.telemetry_host, port=config.telemetry_port, sn=config.adapter_sn)
    await live.connect()
    try:
        adapter = MyAdapter(live=live, connector=runtime.connector)
        await runtime.serve(adapter)
    finally:
        await live.close()
```
