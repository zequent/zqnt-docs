# Zequent Client SDK (Python) — Connector

`client.connector` gives your application direct access to the platform's system of record: asset lookups, organization info, scheduler management, technical configuration, operational policies, and asset payloads.

Full method-by-method reference: [Connector API Reference](../api-reference/client-sdk-connector-python.md).

For Java, see [CONNECTOR.md](CONNECTOR.md).

```python
async with ZequentClient.from_env() as client:
    asset = await client.connector.get_asset_by_sn("YOUR_DEVICE_SN")
```

## Looking up an asset

```python
asset = await client.connector.get_asset_by_id("550e8400-e29b-41d4-a716-446655440000")
```

Asset payload storage, organization lookup, and scheduler CRUD follow the same pattern — see the
[reference](../api-reference/client-sdk-connector-python.md) for the full method list.

**There are no Mission or Task methods on this client** — Python's Connector doesn't cover
mission/task record CRUD.

## Scheduler example

```python
from client_sdk.models import SchedulerDTO

scheduler = SchedulerDTO(name="Nightly patrol", ...)
created = await client.connector.create_scheduler(scheduler)

await client.connector.update_scheduler(created.id, updated_scheduler)
await client.connector.delete_scheduler(created.id)
```

## Capabilities

**Not available in the Python Client SDK.** Confirmed against `RemoteControlClient`'s real source —
there is no `get_capabilities` method (or any capability-related method) anywhere in the Python
SDK, unlike Java's `client.remoteControl().getCapabilities(sn)` and Go's `rc.GetCapabilities(ctx,
sn)`. If you need capability discovery, call it from Java or Go, or query the platform directly.
What an asset reports comes from its edge adapter — see
[Edge SDK (Python) — Connector](../edge-sdk/edge-sdk-python-connector.md#capabilities).

## Error handling

Three different conventions live on this one client — confirmed against the real source, not the
same for every method group:

- **Asset/payload/organization/policy/technical-config methods** (`get_asset_by_sn`,
  `register_asset`, `get_active_policies_by_type`, ...) return the raw DTO/list on success and
  **raise `client_sdk.exceptions.ConnectorError`** on a platform-side (business) error — not
  `grpc.aio.AioRpcError`. A transport failure (network down, deadline exceeded) still raises
  `grpc.aio.AioRpcError` separately; catch both if you need to distinguish "the platform said no"
  from "couldn't reach the platform."
- **Scheduler methods** (`get_scheduler`, `create_scheduler`, ...) never raise for a business
  error — they return a `SchedulerResponse` with `.success`/`.error` populated instead, the same
  convention `client.mission_autonomy` uses throughout (see
  [Mission Autonomy — Error handling](MISSION_AUTONOMY_PYTHON.md#error-handling)). Only a transport
  failure raises, and only as `grpc.aio.AioRpcError`.

```python
import grpc
from client_sdk.exceptions import ConnectorError

try:
    asset = await client.connector.get_asset_by_sn("DOCK-1")
except ConnectorError as e:
    # Platform-side error, e.g. not found — e.error_code / e.error_message carry the detail.
    asset = None
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.UNAVAILABLE:
        # transient — retry/backoff
        raise
    raise

# Scheduler methods: check .success instead of catching ConnectorError.
response = await client.connector.get_scheduler("scheduler-uuid")
if not response.success:
    print(response.error.error_message)
```

## See also

- [Connector API Reference](../api-reference/client-sdk-connector-python.md) — every method
- [Functional Responses](FUNCTIONAL_RESPONSES_PYTHON.md) — what a response actually confirms
