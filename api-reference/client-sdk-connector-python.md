# Zequent Client SDK (Python) — Connector API Reference

Exhaustive method reference for `client.connector`. For a narrative introduction and worked
examples, see the [Connector guide](../client-sdk/CONNECTOR_PYTHON.md). For Java, see
[client-sdk-connector.md](client-sdk-connector.md).

Every method is `async`. On success each returns the unwrapped DTO (or `list`/`None` where noted)
directly — there is no `hasErrors`/`success` field to check here. A platform-side error raises
`grpc.aio.AioRpcError`; see [Connector guide — Error handling](../client-sdk/CONNECTOR_PYTHON.md#error-handling).

**No Mission or Task methods exist on this client.** Python's Connector only covers assets,
payloads, organization, schedulers, and technical config/policies — mission/task record CRUD isn't
part of this SDK's surface.

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_asset_by_sn(sn)` | asset DTO | Look up an asset by serial number |
| `get_asset_by_id(asset_id)` | asset DTO | Look up an asset by platform ID |
| `get_sub_asset_by_sn(sn)` | sub-asset DTO | Look up a sub-asset (e.g. the drone paired to a dock) |
| `register_asset(asset)` | asset DTO | Register a new asset (normally done by an edge adapter, not a customer app) |
| `update_asset(asset_id, asset, update_mask=None)` | asset DTO | Update asset metadata |
| `update_sub_asset(sub_asset_id, sub_asset, update_mask=None)` | sub-asset DTO | Update sub-asset metadata |
| `deregister_asset(sn)` | `None` | Remove an asset |

## Asset payloads

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsert_asset_payload(payload, asset_id=None, sub_asset_id=None, sub_asset_sn=None, update_mask=None)` | payload DTO | Create or update a payload (`payload` is an `AssetPayloadProtoDTO`; exactly one of `asset_id`/`sub_asset_id` is required; empty `update_mask` means a full upsert) |
| `list_asset_payloads(asset_id=None, sub_asset_id=None)` | `list` | List payloads for an asset |
| `delete_asset_payload(payload_id, asset_id=None, sub_asset_id=None)` | payload DTO | Remove a payload |

## Organizations

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_organization(bind_code=None)` | org DTO | Get organization info |

## Schedulers

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_scheduler(scheduler_id)` | `SchedulerResponse` | Get a scheduler by ID |
| `create_scheduler(scheduler)` | `SchedulerResponse` | Create a scheduler (`scheduler` is a `SchedulerDTO`) |
| `create_schedulers(schedulers)` | `SchedulerResponse` | Create several schedulers in one call (`list[SchedulerDTO]`) |
| `update_scheduler(scheduler_id, scheduler)` | `SchedulerResponse` | Update a scheduler |
| `delete_scheduler(scheduler_id)` | `SchedulerResponse` | Delete a scheduler |
| `delete_schedulers(scheduler_ids)` | `SchedulerResponse` | Delete several schedulers in one call |

Note: unlike the asset/payload/organization methods above, the scheduler methods return a
`SchedulerResponse` wrapper rather than a raw DTO — check it the same way as the Java SDK's
scheduler responses.

## Technical configuration & policies

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_active_policies_by_type(policy_type)` | `list` | Fetch active operational policies of a given type |
| `get_all_active_policies()` | `list` | Fetch every active operational policy |
| `get_technical_configs(scope=None, scope_target=None)` | `list` | Fetch technical configuration values for a scope |

## Capabilities

Capability discovery lives on `client.remote_control`, not `client.connector` — see
[Remote Control — Capabilities](../client-sdk/REMOTE_CONTROL.md#capabilities--custom-commands) (Java
page; the Python client mirrors the same `get_capabilities(sn)` call). What an asset reports comes
from its edge adapter — see
[Edge SDK (Python) — Connector](../edge-sdk/edge-sdk-python-connector.md#capabilities).
