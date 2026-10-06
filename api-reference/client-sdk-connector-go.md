# Zequent Client SDK (Go) — Connector API Reference

Method reference for `connector.New(conn)` (package `github.com/Zequent/zqnt-client-sdk-go/v2/connector`).
For a narrative introduction, see the [Connector guide](../client-sdk/CONNECTOR_GO.md). For Java, see
[client-sdk-connector.md](client-sdk-connector.md); for Python,
[client-sdk-connector-python.md](client-sdk-connector-python.md).

Every method takes a `context.Context` first and returns `(result, error)`. There is no separate
`HasErrors` flag to check: a platform-side error (e.g. no asset with that serial number) comes back
as a non-nil `error` carrying the platform's message, and so does a transport failure. A refusal of
the credential keeps its gRPC code, so `status.Code(err)` returns `codes.Unauthenticated` or
`codes.PermissionDenied`.

The Go connector package covers asset lookup by serial number and the Skill Registry. Updating
assets and their payloads is available in the [Java](client-sdk-connector.md) and
[Python](client-sdk-connector-python.md) client SDKs.

## What a client credential may call

A customer application authenticates with a client credential (`ZQNT_CLIENT_TOKEN`, sent by
`auth.DialOptions`). The platform binds it to one organization:

- A method marked **refused** returns an error with `codes.PermissionDenied`.
- Looking up an asset that is not your organization's is refused the same way. An asset that does
  not exist gets the same answer.

## Assets

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `GetAssetBySn(ctx, sn)` | `*asset.AssetProtoDTO` | allowed | Look up an asset by serial number |

## Skill Registry

The platform-wide catalog of every command an edge adapter has reported, one entry per
`(command_id, schema_version)`. Types come from `github.com/Zequent/zqnt-client-sdk-go/v2/gen/connector/proto`;
`SkillContractStatus` is `ACTIVE`, `DRAFT`, `DEPRECATED` or `RETIRED`.

| Method | Returns | Client credential | Purpose |
| --- | --- | --- | --- |
| `ListSkillContracts(ctx, status, commandID)` | `[]*SkillContractProtoDTO` | allowed | List the registry. `status` (`*SkillContractStatus`, `nil` for all) filters by lifecycle state; a non-empty `commandID` returns that command's full version history instead |
| `ObserveSkillContract(ctx, contract)` | `*SkillContractProtoDTO` | refused | Record a contract (done by the platform when adapters report capabilities) |
| `SetSkillContractStatus(ctx, id, status)` | `*SkillContractProtoDTO` | refused | Change a contract's lifecycle state |
| `SetSkillContractPermissions(ctx, id, requiredPermissions)` | `*SkillContractProtoDTO` | refused | Replace a contract's required permissions |

## Capabilities

What a specific asset can do right now is read from the Remote Control client:
`rc.GetCapabilities(ctx, sn)` — see [client-sdk-remote-control-go.md](client-sdk-remote-control-go.md).
