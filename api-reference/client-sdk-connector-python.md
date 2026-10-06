# Zequent Client SDK (Python) — Connector API Reference

Method reference for `client.connector`. For a narrative introduction and worked examples, see the
[Connector guide](../client-sdk/CONNECTOR_PYTHON.md). For Java, see
[client-sdk-connector.md](client-sdk-connector.md); for Go,
[client-sdk-connector-go.md](client-sdk-connector-go.md).

Every request/response method is `async` and works with the generated proto types
(`AssetProtoDTO`, `SubAssetProtoDTO`, `AssetPayloadProtoDTO`, `SkillContractProtoDTO`, ...). On
success it returns the DTO (or a `list`, or `None` where noted). Errors:

- The platform answered with an error (e.g. not found): `client_sdk.exceptions.ConnectorError`
  (`operation`, `error_code`, `error_message`).
- The platform refused the credential: `client_sdk.auth.ZequentAuthError`, a
  `grpc.aio.AioRpcError` with code `UNAUTHENTICATED` or `PERMISSION_DENIED` and a message that says
  what to do.
- A transient failure that outlasted every retry: `client_sdk.ZequentRetryExhaustedError`. Other
  gRPC failures raise `grpc.aio.AioRpcError`.

## What a client credential may call

A customer application authenticates with a client credential (`ZQNT_CLIENT_TOKEN`). The platform
binds it to one organization and allows only part of this client. Everything else is administration
(done in the Admin Console) or belongs to edge adapters and the platform's own services.

- A method marked **refused** raises `ZequentAuthError` with `PERMISSION_DENIED`.
- A method that names an asset (by serial number or ID) which is not your organization's is refused
  the same way. An asset that does not exist gets the same answer.

## Assets

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `get_asset_by_sn(sn)` | `AssetProtoDTO` | allowed | Look up an asset by serial number |
| `get_asset_by_id(asset_id)` | `AssetProtoDTO` | allowed | Look up an asset by platform ID |
| `get_sub_asset_by_sn(sn)` | `SubAssetProtoDTO` | allowed | Look up a sub-asset (e.g. the drone paired to a dock) |
| `update_asset(asset_id, asset, update_mask=None)` | `AssetProtoDTO` | allowed | Update asset metadata. Without `update_mask`, every mutable field is replaced |
| `update_sub_asset(sub_asset_id, sub_asset, update_mask=None)` | `SubAssetProtoDTO` | allowed | Update sub-asset metadata |
| `deregister_asset(sn)` | `None` | allowed | Remove an asset |
| `register_asset(asset)` | `AssetProtoDTO` | refused | Assets are added in the Admin Console or paired by their edge adapter with a pairing code |

## Asset payloads

Hardware mounted on an asset or sub-asset (a camera, gimbal or sensor): slot, name, serial number,
kind, vendor, model, firmware version and state. Pass exactly one of `asset_id` / `sub_asset_id` as
the owner.

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `upsert_asset_payload(payload, asset_id=None, sub_asset_id=None, sub_asset_sn=None, update_mask=None)` | `AssetPayloadProtoDTO` | allowed | Create or update a payload. Without `update_mask`, it is a full upsert |
| `list_asset_payloads(asset_id=None, sub_asset_id=None)` | `list` | allowed | List an owner's payloads |

## Skill Registry

The platform-wide catalog of every command an edge adapter has reported, one entry per
`(command_id, schema_version)`. `status` is a `SkillContractStatus` value (`ACTIVE`, `DRAFT`,
`DEPRECATED`, `RETIRED`) from `zqnt_utils.generated.zqnt.connector_pb2`.

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `list_skill_contracts(status=None, command_id=None)` | `list` | allowed | List the registry. `status` filters by lifecycle state; `command_id` returns that command's full version history instead |
| `observe_skill_contract(contract)` | `SkillContractProtoDTO` | refused | Record a contract (done by the platform when adapters report capabilities) |
| `set_skill_contract_status(contract_id, status)` | `SkillContractProtoDTO` | refused | Change a contract's lifecycle state |
| `set_skill_contract_permissions(contract_id, required_permissions)` | `SkillContractProtoDTO` | refused | Replace a contract's required permissions |

## Administration (refused for a client credential)

These exist on the client, but a client credential gets `PERMISSION_DENIED`. Manage them in the
Admin Console.

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_organization(bind_code=None)` | org DTO | Organization info |
| `get_scheduler` / `create_scheduler` / `create_schedulers` / `update_scheduler` / `delete_scheduler` / `delete_schedulers` | `SchedulerResponse` | Schedules that run an Application — see [Applications & Skills](../concepts/applications-and-skills.md) |
| `get_active_policies_by_type(policy_type)` / `get_all_active_policies()` | `list` | Operational policies (asset selection) |
| `get_technical_configs(scope=None, scope_target=None)` | `list` | Technical configuration values — see [Configuration](../concepts/configuration.md) |

## Batch ingestion (refused for a client credential)

`store_telemetry_batch()`, `store_detection_batch()` and `store_notification_batch()` open a
client-streaming `ConnectorBatchSession`. They are the platform's own persistence path for data
coming from edge adapters, and a client credential is refused. Edge adapters report telemetry,
detections and command execution events through the Edge SDK — see
[Live Data (Python)](../edge-sdk/edge-sdk-python-live-data.md).

## Capabilities

The Python client has no capability lookup. Read what an asset can do from the Java
(`client.remoteControl().getCapabilities(sn)`) or Go (`rc.GetCapabilities(ctx, sn)`) client SDK —
see [Remote Control — Capabilities & custom commands](../client-sdk/REMOTE_CONTROL.md#capabilities--custom-commands).
What an asset reports comes from its edge adapter — see
[Edge SDK (Python) — Connector](../edge-sdk/edge-sdk-python-connector.md#capabilities-and-the-skill-registry).
