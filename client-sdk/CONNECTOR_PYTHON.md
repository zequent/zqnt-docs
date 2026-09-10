# Zequent Client SDK (Python) — Connector

`client.connector` gives your application direct access to the platform's system of record: asset lookups, organization info, scheduler management, technical configuration, operational policies, and asset payloads.

For Java, see [CONNECTOR.md](CONNECTOR.md).

```python
async with ZequentClient.from_env() as client:
    asset = await client.connector.get_asset_by_sn("YOUR_DEVICE_SN")
```

## Assets

| Method | Purpose |
| --- | --- |
| `get_asset_by_sn(sn)` | Look up an asset by serial number |
| `get_asset_by_id(asset_id)` | Look up an asset by platform ID |
| `get_sub_asset_by_sn(sn)` | Look up a sub-asset (e.g. the drone paired to a dock) |
| `register_asset(asset)` | Register a new asset (normally done by an edge adapter, not a customer app) |
| `update_asset(asset_id, asset, update_mask=None)` | Update asset metadata |
| `update_sub_asset(sub_asset_id, sub_asset, update_mask=None)` | Update sub-asset metadata |
| `deregister_asset(sn)` | Remove an asset |

```python
asset = await client.connector.get_asset_by_id("550e8400-e29b-41d4-a716-446655440000")
```

## Asset payloads

```python
await client.connector.upsert_asset_payload(asset_id="...", key="flight-plan", data=b"...")
payloads = await client.connector.list_asset_payloads(asset_id="...")
await client.connector.delete_asset_payload(payload_id="...")
```

## Organizations

```python
org = await client.connector.get_organization()
```

## Schedulers

Schedulers define when and how often a task or command runs.

```python
from client_sdk.models import SchedulerDTO

scheduler = SchedulerDTO(name="Nightly patrol", ...)
created = await client.connector.create_scheduler(scheduler)

await client.connector.update_scheduler(created.id, updated_scheduler)
await client.connector.delete_scheduler(created.id)
```

## Technical configuration & policies

```python
configs = await client.connector.get_technical_configs(scope="ORGANIZATION")
policies = await client.connector.get_active_policies_by_type("GEOFENCE")
all_policies = await client.connector.get_all_active_policies()
```

## Capabilities

Ask what an asset supports before offering it as an option:

```python
snapshot = await client.remote_control.get_capabilities(sn)
for capability in snapshot.capabilities:
    print(capability.command_id, capability.state)
```

What an asset reports comes from its edge adapter — see
[Edge SDK (Python) — Connector](../edge-sdk/edge-sdk-python-connector.md#capabilities).

## Error handling

```python
import grpc

try:
    asset = await client.connector.get_asset_by_sn("DOCK-1")
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.NOT_FOUND:
        asset = None
    else:
        raise
```
