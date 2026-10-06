# Zequent Client SDK (Python) — Connector

`client.connector` gives your application access to the platform's system of record: your organization's assets, the payloads mounted on them, and the Skill Registry. A client credential (`ZQNT_CLIENT_TOKEN`) reaches only its own organization's assets. Schedules, policies, technical configuration and organizations are administration: they are managed in the Admin Console, and a client credential is refused (`PERMISSION_DENIED`) — see [what a client credential may call](../api-reference/client-sdk-connector-python.md#what-a-client-credential-may-call).

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

Updating an asset, and listing or updating the payloads mounted on it (camera, gimbal, sensor),
follow the same pattern — see the [reference](../api-reference/client-sdk-connector-python.md) for
the full method list.

**There are no Mission or Task methods on this client** — missions and tasks are gone in 2.0;
automated work is built as Applications and Skills (see
[Applications & Skills](../concepts/applications-and-skills.md)).

## Skill Registry

The Skill Registry is the platform's catalog of every command an edge adapter has reported, one
entry per `(command_id, schema_version)`, each with a lifecycle status:

```python
for c in await client.connector.list_skill_contracts():
    print(c.command_id, c.schema_version, c.status)
```

## Capabilities

**Not available in the Python Client SDK.** Confirmed against `RemoteControlClient`'s real source —
there is no `get_capabilities` method (or any capability-related method) anywhere in the Python
SDK, unlike Java's `client.remoteControl().getCapabilities(sn)` and Go's `rc.GetCapabilities(ctx,
sn)`. If you need capability discovery, call it from Java or Go, or query the platform directly.
What an asset reports comes from its edge adapter — see
[Edge SDK (Python) — Connector](../edge-sdk/edge-sdk-python-connector.md#capabilities-and-the-skill-registry).

## Error handling

Connector methods return the DTO on success and raise otherwise:

- `client_sdk.exceptions.ConnectorError` — the platform answered with an error (e.g. not found);
  `error_code` / `error_message` carry the detail.
- `client_sdk.auth.ZequentAuthError` — the platform refused the credential (`UNAUTHENTICATED`, or
  `PERMISSION_DENIED` for something a client credential may not do). It is a
  `grpc.aio.AioRpcError`; `details()` says what to do.
- `client_sdk.ZequentRetryExhaustedError` — a transient failure (network down, platform
  unavailable) that outlasted every retry. Other gRPC failures raise `grpc.aio.AioRpcError`.

```python
import grpc
from client_sdk.exceptions import ConnectorError

try:
    asset = await client.connector.get_asset_by_sn("DOCK-1")
except ConnectorError as e:
    print("Platform said no:", e.error_code, e.error_message)
except grpc.aio.AioRpcError as e:
    print("Call failed:", e.code(), e.details())
```

## See also

- [Connector API Reference](../api-reference/client-sdk-connector-python.md) — every method
- [Functional Responses](FUNCTIONAL_RESPONSES_PYTHON.md) — what a response actually confirms
