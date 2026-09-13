# Zequent Client SDK (Go) — Connector API Reference

Exhaustive method reference for `connector.New(conn)`. For a narrative introduction and worked
examples, see the [Connector guide](../client-sdk/QUICKSTART_GO.md#connector--assets-schedulers-policies-config).
For Java, see [client-sdk-connector.md](client-sdk-connector.md); for Python,
[client-sdk-connector-python.md](client-sdk-connector-python.md).

Every method takes a `context.Context` first and returns `(*Result, error)` — there's no separate
`hasErrors` flag to check; a non-nil `error` already carries the platform-side message. This is the
**narrowest** of the three Connector surfaces: no asset payloads, organizations, asset
register/update/deregister, missions, or tasks.

## Assets

| Method | Purpose |
| --- | --- |
| `GetAssetBySn(ctx, sn)` | Look up an asset by its serial number |

Read-only — registration/update is normally done by an edge adapter, not a customer app.

## Schedulers

`ConnectorService` has no `ListSchedulers` RPC at this contract version — listing lives on
`missionautonomy.New(conn)` instead: `ma.ListSchedulers(ctx, taskID)` (empty `taskID` = unfiltered).
See [Mission Autonomy — Quickstart](../client-sdk/QUICKSTART_GO.md#missionautonomy--missions-tasks--schedulers).

| Method | Purpose |
| --- | --- |
| `GetScheduler(ctx, schedulerID)` | Get a scheduler by ID |
| `CreateScheduler(ctx, scheduler)` | Create one scheduler |
| `CreateSchedulers(ctx, schedulers)` | Create several in one call |
| `UpdateScheduler(ctx, schedulerID, scheduler)` | Update a scheduler |
| `DeleteScheduler(ctx, schedulerID)` | Delete one scheduler |
| `DeleteSchedulers(ctx, schedulerIDs)` | Delete several in one call |
| `DeleteSchedulersByTask(ctx, taskID)` | Delete every scheduler tied to one task |

## Technical configuration & policies

| Method | Purpose |
| --- | --- |
| `GetActivePoliciesByType(ctx, policyType)` | Fetch active operational policies of a given type |
| `GetAllActivePolicies(ctx)` | Fetch every active operational policy |
| `GetTechnicalConfigs(ctx, scope, scopeTarget)` | Fetch technical configuration values for a scope |

## Capabilities

Capability discovery lives on `remotecontrol.New(conn)`, not `connector.New(conn)` — see
`rc.GetCapabilities(ctx, sn)` in the [Quickstart](../client-sdk/QUICKSTART_GO.md#remotecontrol--manual-flightdock-control-gateway).
