# Edge SDK (Python) — Connector

`ConnectorClient` gives an edge adapter access to the platform's asset registry over gRPC — what an
adapter itself needs: making sure its asset exists (pairing it with a one-time code if needed),
looking assets up, and reporting its commands to the Skill Registry. The 1.3 Mission and Task methods
are gone: work reaches an adapter as commands (see
[Edge Adapter — Custom commands](edge-sdk-python-adapter.md#custom-commands)).

Full method-by-method reference, including retry/timeout behavior: [Connector API Reference](../api-reference/edge-sdk-python-connector-reference.md).

For Java, see [edge-sdk-connector.md](edge-sdk-connector.md).

---

## Lifecycle

`EdgeAdapterRuntime` connects a `ConnectorClient` for you, with the adapter's edge credential and
`ZQNT_CLAIM_CODE` — use `runtime.connector` (see the [Quickstart](edge-sdk-python-quickstart.md)). To
use one on its own:

```python
from edge_sdk import ConnectorClient

conn = ConnectorClient(host="localhost", port=8010)   # token defaults to ZQNT_EDGE_TOKEN
await conn.connect()
try:
    asset = await conn.get_asset_by_sn("DOCK-1")
finally:
    await conn.close()
```

---

## Pairing the asset

An adapter does not create assets. Its asset is either created in the Admin Console, or **paired**
with a one-time pairing code (Admin Console, Assets page, **Pairing codes**), set as
`ZQNT_CLAIM_CODE`. `ensure_asset(asset)` covers both: it looks the serial number up, and redeems the
code only if the asset is unknown — a code is single-use, so after the first pairing there is nothing
left to redeem. Without a code it creates nothing.

- `redeem_asset_claim(code, asset)` trades a code for an asset directly. The organization the asset
  lands in comes from the code, never from `asset`. It returns `None` for every refusal alike —
  unknown, expired, revoked, exhausted, or not valid for this kind of device.
- `register_asset` no longer exists.

## Asset lookup

```python
asset = await conn.get_asset_by_sn("DOCK-1")
if asset is None:
    log.warning("Asset not found")
```

---

## Capabilities and the Skill Registry

An adapter reports which commands it supports by implementing `get_capabilities(sn, asset_id)` on
`EdgeAdapter` — normally through `register_command` and `_auto_capabilities` (see
[Edge Adapter](edge-sdk-python-adapter.md#custom-commands)). The platform calls it when it needs to know
what an asset can do — for example so the Admin Console can hide controls an asset does not support,
rather than failing at execution time.

Beyond that live snapshot, `observe_skill_contract`, `list_skill_contracts`,
`set_skill_contract_status` and `set_skill_contract_permissions` report and manage the commands in the
platform's persisted Skill Registry — see the
[reference](../api-reference/edge-sdk-python-connector-reference.md#skill-registry).

## Error handling

The asset and Skill Registry methods return `None` (or `[]`) on a business-level failure (not found,
refused) rather than raising — check for `None` explicitly. Transport-level failures (`UNAVAILABLE`,
timeouts) raise `grpc.aio.AioRpcError`:

```python
import grpc

try:
    asset = await conn.get_asset_by_sn("DOCK-99")
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.UNAVAILABLE:
        log.warning("Connector service unreachable; retrying")
    raise
```

---

## Typical startup pattern

```python
import asyncio
import logging

from edge_sdk import Asset, AssetConnection, AssetType, AssetVendor, EdgeAdapterConfig

from .adapter import MyDeviceAdapter

log = logging.getLogger(__name__)


async def main():
    config = EdgeAdapterConfig.from_env()
    async with config.runtime() as runtime:
        # Bind to the asset if the platform knows it; otherwise redeem ZQNT_CLAIM_CODE for it
        asset = await runtime.connector.ensure_asset(Asset(
            id=None,
            sn=config.adapter_sn,
            name="Roof Dock 1",
            type=AssetType.DOCK,
            vendor=AssetVendor.DJI,
            connection=AssetConnection.TCP,
            model="",
            organization="",   # decided by the pairing code, never by the adapter
        ))
        if asset is None:
            log.warning("Asset %s is not on the platform yet: create it in the console or set a pairing code",
                        config.adapter_sn)
        await runtime.serve(MyDeviceAdapter())


if __name__ == "__main__":
    asyncio.run(main())
```
