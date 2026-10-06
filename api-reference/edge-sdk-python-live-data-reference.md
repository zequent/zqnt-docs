# Edge SDK (Python) — Live Data API Reference

> For 1.3.x (end of life), see the [1.3 Live Data reference](edge-sdk-python-live-data-reference-1.3.md).

Method reference for `LiveDataService`, the facade over `TelemetryPublisher`,
`DetectionPublisher`, and `NotificationPublisher`. For a narrative introduction and worked examples,
see the [Live Data guide](../edge-sdk/edge-sdk-python-live-data.md). For Java, see
[edge-sdk-live-data.md](edge-sdk-live-data-reference.md).

`LiveDataService(host, port=50052, sn="", queue_max_size=1000)` — `port` defaults to `50052`, which
is **not** the Live Data Service's real platform port (`8003`, per the
[image table](../README.md#platform-service-images)); always pass `port` explicitly. `queue_max_size`
is forwarded to all three underlying publishers — see [Reconnection and queueing behavior](#reconnection-and-queueing-behavior).

## Lifecycle

| Method | Returns | Purpose |
| --- | --- | --- |
| `connect()` | `None` | Starts all three background streaming tasks. Returns almost immediately — it does not wait for the platform to actually be reachable |
| `close()` | `None` | Drains and stops all three streams. Safe to call even if some never successfully connected |
| `__aenter__()` / `__aexit__()` | — | `connect()`/`close()` as an `async with` context manager |

## Telemetry

| Method | Parameter | Purpose |
| --- | --- | --- |
| `produce_telemetry(telemetry)` | `AssetTelemetry \| SubAssetTelemetry` | Routes to `publish_asset_telemetry`/`publish_subasset_telemetry` internally based on the type passed |

Field reference: [Models Reference — Telemetry](edge-sdk-python-models.md#telemetry).

## Detection

| Method | Parameter | Purpose |
| --- | --- | --- |
| `produce_detection(batch)` | `DetectionBatch` | Routes to `publish_detection_batch` internally |

## Notifications

| Method | Parameter | Purpose |
| --- | --- | --- |
| `produce_notification(event)` | `CommandExecutionEvent \| AssetStatusEvent \| MissionEvent` | Routes to `publish_command_execution_event`/`publish_asset_status`/`publish_mission_event` based on the type passed; raises `TypeError` for any other type |

| Event class | Fields | Use |
| --- | --- | --- |
| `CommandExecutionEvent` | `external_execution_id`, `status`, `sn`, `command_id`, `progress`, `message`, `output`, `occurred_at` | The outcome of a command you accepted — this is what moves a Skill run on |
| `AssetStatusEvent` | `sn`, `online`, `asset_id`, `message` | The asset went online or offline |
| `MissionEvent` | `mission_id`, `mission_type`, `status`, `sn`, `message` | Informational; stored as a notification |

`CommandExecutionEvent`:

| Field | Notes |
| --- | --- |
| `external_execution_id` | The execution id you returned with the accepted response (or `ctx.tid` if you returned none) — it is how the platform finds the run |
| `status` | `CommandExecutionStatus.ACCEPTED`, `RUNNING`, `SUCCEEDED`, `FAILED` or `CANCELLED` |
| `sn` | The asset that ran the command — always set it |
| `command_id` | The command id, e.g. `mission.waypoint.execute` |
| `progress` | 0.0–1.0, while `RUNNING` |
| `output` | `dict` result on `SUCCEEDED`, available to later nodes as `$.nodes.<id>.output.<field>` |
| `occurred_at` | When it happened; defaults to the publish time |

## Reconnection and queueing behavior

Each of the three underlying publishers (`TelemetryPublisher`, `DetectionPublisher`,
`NotificationPublisher`) manages its own stream independently — a detection-stream failure doesn't
interrupt telemetry. Confirmed identical across all three:

- **Backoff**: starts at 1.0s, doubles on each reconnect attempt, capped at 60.0s. Resets to 1.0s
  only after a stream completes cleanly (a clean server-side close), not merely after successfully
  reconnecting.
- **No attempt limit** — reconnection continues indefinitely.
- **`produce_*`/`publish_*` calls never raise for a disconnected stream.** Each call enqueues onto
  an internal `asyncio.Queue` (`maxsize=queue_max_size`, default 1000) and returns immediately. If
  the queue is already full, the **new** item is dropped (not enqueued) and the drop is logged at
  `debug` level — nothing is raised, and no warning-level log is emitted either. This differs from
  what "older frames are dropped" might suggest: already-queued items are left alone; only an
  incoming call made while the queue is full is the one that's lost.
- **`connect()` does not verify connectivity.** It only starts the background reconnect-loop task
  and returns — call it once at startup even if the platform isn't reachable yet.
