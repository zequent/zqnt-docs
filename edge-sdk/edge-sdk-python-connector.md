# Edge SDK (Python) — Connector

`ConnectorClient` gives an edge adapter access to the platform's asset registry over gRPC — what an adapter itself needs (registering its own asset, watching asset state, reporting supported commands). It also exposes `get_mission`, `get_task` and `get_task_by_flight_id`, for an adapter to resolve a task the platform asked it to run — but checked against all five real Python adapters (MAVLink, Sapient, AI, Betaflight, RNS), **none of them actually call these three methods**. Every real one drives its mission/task work entirely through `EdgeAdapter.send_custom_command`/`prepare_task`/`start_task` instead (see [MAVLink](edge-sdk-mavlink-adapter-deployment.md) for a concrete example), with no dependency on resolving a task via `ConnectorClient`. Creating and managing missions and tasks belongs to the **Client SDK**, used by customer applications.

Full method-by-method reference, including retry/timeout behavior: [Connector API Reference](../api-reference/edge-sdk-python-connector-reference.md).

For Java, see [edge-sdk-connector.md](edge-sdk-connector.md).

---

## Lifecycle

```python
from edge_sdk import ConnectorClient

conn = ConnectorClient(host="localhost", port=8010)
await conn.connect()
try:
    asset = await conn.get_asset_by_sn("DOCK-1")
finally:
    await conn.close()
```

---

## Asset registration

```python
from edge_sdk import Asset, AssetType, AssetVendor

asset = Asset(
    sn="DOCK-1",
    name="Roof Dock 1",
    type=AssetType.DOCK,
    vendor=AssetVendor.DJI,
)

asset_id = await conn.register_asset(asset)
if asset_id is None:
    log.error("Asset registration failed")
```

## Asset lookup

```python
asset = await conn.get_asset_by_sn("DOCK-1")
if asset is None:
    log.warning("Asset not found")
```

## Watching asset state

Subscribe to the platform's asset-monitoring stream — useful if your adapter process needs to react to changes made elsewhere (e.g. through the Admin Console):

```python
async for assets in conn.watch_assets():
    for asset in assets:
        log.info("Asset update: %s -> %s", asset.sn, asset.status)
```

The stream runs until cancelled or the server closes it; wrap it in your own retry loop if you want automatic reconnection.

---

## Capabilities

An adapter reports which commands it supports by implementing `get_capabilities(sn, asset_id)` on
`EdgeAdapter`, returning a `Capabilities` object. The platform calls this when it needs to know what
an asset can do — for example so the Admin Console can hide controls an asset does not support,
rather than failing at execution time.

Return an empty set for an asset you do not recognise. See
[Edge Adapter](edge-sdk-python-adapter.md) for the full `EdgeAdapter` surface.

> **Beta preview — 2.0.x, not yet released.** An unmerged branch adds four Skill Registry methods
> to `ConnectorClient` (`observe_skill_contract`, `list_skill_contracts`,
> `set_skill_contract_status`, `set_skill_contract_permissions`) that let an adapter push its own
> command contracts into a persisted registry directly, instead of only ever being polled
> indirectly through `get_capabilities` above. It also removes `get_mission`/`get_task`/
> `get_task_by_flight_id` from this client outright. None of this is on `main`/the current 1.3.x
> release yet — see the
> [2.0.x Beta Connector reference](../api-reference/edge-sdk-python-connector-reference-2.0.md) if
> you want to see where this is headed.

## Error handling

`get_asset_by_sn` and `register_asset` return `None` on a business-level failure (not found / registration rejected) rather than raising — check for `None` explicitly. Transport-level failures (`UNAVAILABLE`, timeouts) raise `grpc.aio.AioRpcError`:

```python
import grpc

try:
    asset = await conn.get_asset_by_sn("DOCK-99")
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.UNAVAILABLE:
        log.warning("Connector service unreachable; retrying")
    raise
else:
    if asset is None:
        asset_id = await conn.register_asset(default_asset)
```

---

## Typical startup pattern

```python
async def boot(adapter):
    conn = ConnectorClient(host=os.environ["CONNECTOR_SERVICE_HOST"], port=int(os.environ.get("CONNECTOR_SERVICE_PORT", 8010)))
    await conn.connect()
    asset = await conn.get_asset_by_sn(os.environ["ZEQUENT_EDGE_SN"])
    if asset is None:
        asset_id = await conn.register_asset(default_asset_from_env())
    # hand `conn` to your adapter however it expects it -- EdgeAdapterRuntime does this
    # wiring for you when you use it (see the Quickstart).
```
