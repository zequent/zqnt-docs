# Edge SDK (Python) — Connector API Reference

> **Beta preview:** an unmerged 2.0.x branch adds Skill Registry self-reporting to this client and
> removes `get_mission`/`get_task`/`get_task_by_flight_id` outright — see the
> [2.0.x Beta reference](edge-sdk-python-connector-reference-2.0.md) if you want to see where this
> is headed. Not on `main`/the current 1.3.x release yet.

Exhaustive method reference for `ConnectorClient`. For a narrative introduction and worked examples,
see the [Connector guide](../edge-sdk/edge-sdk-python-connector.md). For Java, see
[edge-sdk-adapter.md](edge-sdk-adapter-reference.md) / [edge-sdk-connector.md](edge-sdk-connector-reference.md).

`ConnectorClient(host, port=50053, call_timeout=30.0, max_retries=3)` — `port` defaults to `50053`,
which is **not** the Connector Service's real platform port (`8010`, per the
[image table](../README.md#platform-service-images)); always pass `port` explicitly rather than
relying on the constructor default.

## Lifecycle

| Method | Returns | Purpose |
| --- | --- | --- |
| `connect()` | `None` | Open the gRPC channel and initialize the stub |
| `close()` | `None` | Close the channel and release resources |

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_asset_by_sn(sn)` | `Asset \| None` | Look up an asset by serial number; `None` if not found |
| `register_asset(asset)` | `str \| None` | Register a new asset; returns the asset id, or `None` if the platform rejected it |
| `watch_assets()` | `AsyncIterator[list[Asset]]` | Subscribe to the platform's asset-monitoring stream — yields a snapshot list on every server push; runs until cancelled or the server closes it |

## Missions and tasks

Read-only lookups — there is no `create`/`update`/`delete` on `ConnectorClient` for either. **No
confirmed real-adapter usage**: checked against all five real Python adapters (MAVLink, Sapient, AI,
Betaflight, RNS), none call these three methods — every real adapter drives its mission/task work
through `EdgeAdapter.send_custom_command`/`prepare_task`/`start_task` instead. See the
[guide](../edge-sdk/edge-sdk-python-connector.md) for the full explanation.

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_mission(mission_id, sn="")` | `Mission \| None` | Fetch a mission by ID |
| `get_task(task_id, sn="")` | `Task \| None` | Fetch a task by ID |
| `get_task_by_flight_id(flight_id, sn="")` | `Task \| None` | Fetch a task by its external flight ID |

`Mission`/`Task` field reference: [Models Reference — Tasks and missions](edge-sdk-python-models.md#tasks-and-missions).

## Error handling

`get_asset_by_sn` and `register_asset` (and `get_mission`/`get_task`/`get_task_by_flight_id`) return
`None` on a business-level failure (not found, validation) rather than raising. Transport-level
failures (`UNAVAILABLE`, timeouts, ...) raise `grpc.aio.AioRpcError`.

## Retry and timeout behavior

Every call carries a per-call deadline (`call_timeout`, default 30.0s) and is retried automatically
on `UNAVAILABLE`/`DEADLINE_EXCEEDED` — every other gRPC error propagates immediately, unretried:

- Up to `max_retries` attempts (default 3, set in the constructor).
- Exponential backoff starting at 0.5s, doubling each retry, capped at 10s.
- Each call logs its transaction id (`tid`, a fresh UUID per call) so failures can be correlated
  across services.
