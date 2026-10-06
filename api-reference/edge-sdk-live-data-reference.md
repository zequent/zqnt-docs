# Edge SDK — Live Data Service API Reference

> For 1.3.x (end of life), see the [1.3 Live Data reference](edge-sdk-live-data-reference-1.3.md).

Method reference for `LiveDataService`. For a narrative introduction and worked examples,
see the [Live Data guide](../edge-sdk/edge-sdk-live-data.md).

Every method returns `CompletableFuture<Void>` that completes once the message is queued onto the
device's stream (not once the platform has processed it).

## Telemetry

| Method | Parameters | Description |
|---|---|---|
| `produceTelemetryData(TelemetryRequestData)` | requestData (POJO) | Send telemetry — recommended API |
| `produceTelemetry(String deviceSn, ProduceTelemetryRequest)` | deviceSn, telemetryRequest (Proto) | Send telemetry using the raw Proto message directly |

See [Models Reference — TelemetryData](edge-sdk-models.md#telemetrydata) for the full field list.

## Detections

| Method | Parameters | Description |
|---|---|---|
| `produceDetectionData(DetectionRequestData)` | requestData (POJO) | Send a detection batch — recommended API |
| `produceDetection(String deviceSn, DetectionBatch)` | deviceSn, detectionBatch (Proto) | Send a detection batch using the raw Proto message directly |

`DetectionRequestData`: `tid`, `sn`, `timestamp`, `streamUrl`, `detections` (list of
`DetectionResultData`: `objectId`, `objectType`, `confidence`, `boundingBox`); `BoundingBoxData`:
`x`, `y`, `width`, `height`.

## Notifications

| Method | Parameters | Description |
|---|---|---|
| `produceNotificationData(NotificationRequestData)` | requestData (POJO) | Send a notification — recommended API |
| `produceNotification(String deviceSn, ProduceNotificationRequest)` | deviceSn, notificationRequest (Proto) | Send a notification using the raw Proto message directly |

`NotificationRequestData` carries `tid`, `sn`, `timestamp`, `severity`, `eventType`, and one event
— set exactly one per call:

| Event field | Type | Use |
|---|---|---|
| `commandExecutionEvent` | `CommandExecutionEventData` | The outcome of a command you accepted — this is what moves a Skill run on |
| `assetStatusEvent` | `AssetStatusEventData` (`sn`, `assetId`, `online`, `message`) | The asset went online or offline |
| `missionEvent` | `MissionEventData` (`missionId`, `missionType`, `status`, `message`) | Informational; stored as a notification |

`CommandExecutionEventData`:

| Field | Notes |
|---|---|
| `externalExecutionId` | The execution id you returned in `CommandResult.accepted(...)` (or the request's transaction id if you returned none) — required, it is how the platform finds the run |
| `commandId` | The command id, e.g. `mission.waypoint.execute` |
| `status` | `COMMAND_EXECUTION_STATUS_ACCEPTED`, `_RUNNING`, `_SUCCEEDED`, `_FAILED` or `_CANCELLED` |
| `occurredAt` | When it happened — required |
| `progress`, `message` | Optional progress and text |
| `output` | Optional result, available to later nodes as `$.nodes.<id>.output.<field>` |
| `error` | Why it failed (with `_FAILED`) |
| `assetSn` | The asset that ran the command — required |

`externalExecutionId`, `assetSn` and `occurredAt` are required: the platform drops an event without
them, and the Skill node waits until it times out. A worked example is in the
[Edge Adapter guide — Custom Commands](../edge-sdk/edge-sdk-adapter.md#custom-commands).

## Stream Management

| Method | Parameters | Description |
|---|---|---|
| `closeStream(String deviceSn)` | deviceSn | Close the telemetry stream for one device |
| `closeDetectionStream(String deviceSn)` | deviceSn | Close the detection stream for one device |
| `closeNotificationStream(String deviceSn)` | deviceSn | Close the notification stream for one device |
| `closeAllStreams()` | — | Close every active **telemetry** stream. Despite the name, this does not touch detection or notification streams — call `closeDetectionStream`/`closeNotificationStream` per device for those. |

`LiveDataServiceImpl` additionally exposes a `shutdown()` method (not on the `LiveDataService`
interface) that closes all three stream types for every device and shuts down the internal
reconnect scheduler in one call — this is the one to use for a full, orderly shutdown. See
[Wiring and shutdown](../edge-sdk/edge-sdk-live-data.md#wiring-and-shutdown) for why your adapter
has to call it itself.

## Reconnection behavior

Confirmed against `LiveDataServiceImpl` (each data kind — telemetry, detections, notifications —
tracks its own reconnect state independently):

- **Backoff**: starts at a 2-second delay and doubles on each successive attempt up to a 60-second
  cap, plus up to 25% additional random jitter added on top of that value each time.
- **No attempt limit.** Reconnection continues indefinitely until the stream succeeds or is
  explicitly closed — there is no maximum-attempts cutoff.
- **Attempt counter reset**: the per-device attempt count resets to zero as soon as the platform
  sends any response on the stream, and independently 10 seconds after a (re)opened stream is still
  active — so a later failure starts backoff again from 2 seconds rather than continuing from
  wherever it left off.
- **Some failures are not retried at all.** A stream that fails with gRPC status
  `UNAUTHENTICATED`, `PERMISSION_DENIED`, `FAILED_PRECONDITION`, `UNIMPLEMENTED`, or `DATA_LOSS` is
  treated as permanent — no reconnect is scheduled, and the failure is logged as an error.
