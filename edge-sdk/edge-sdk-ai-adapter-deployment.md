# AI Edge Adapter -- Deployment Guide

The AI Adapter pulls a live RTMP/RTSP video stream, runs YOLO object detection/tracking on it, georeferences each detection against the target camera's live telemetry (turning a bounding box into an approximate lat/lon), publishes the results to the platform, and can optionally re-aim the target's gimbal at whatever it's currently tracking.

Unlike every other adapter, this one doesn't drive the hardware it watches — it registers as its **own** logical asset (`ASSET_TYPE_SENSOR`) and observes a *different* asset's telemetry and video as a client, using the same Live Data streaming and Remote Control gRPC contracts every other adapter and the Client SDK use.

**Status: early access, source-only.** The adapter itself is built on the standard `EdgeAdapterRuntime`/`EdgeAdapterConfig` pattern (the same one every other Python adapter uses), but there's no `Dockerfile` or published image yet — run it from source.

---

## Prerequisites

- Python 3.12+ and [`uv`](https://docs.astral.sh/uv/)
- A YOLO model file (`AI_MODEL_PATH`)
- The RTMP/RTSP stream URL and telemetry of the asset you want to watch
- Running Zequent platform services: Connector Service, Live Data Service, Remote Control Service (only needed if you use gimbal re-aim)

---

## How it works

1. The adapter registers itself as a `SENSOR` asset.
2. A detection session is started via a custom command (see below), pointed at a target asset's video stream.
3. Each detected object's bounding box is converted to an approximate geographic position using the camera's field of view and the target's live telemetry (`StreamTelemetry`).
4. Results are published via `DetectionPublisher` / `ProduceDetection`, visible the same way any other adapter's detections are.
5. If enabled, the adapter calls `RemoteControlService.LookAt` to re-aim the target's gimbal at the tracked object.

## Detection control (custom command)

Starting, stopping, and toggling tracking is done through `SendCustomCommand` with `command_type = "detection.control"` — the same generic custom-command pattern other adapters use (e.g. MAVLink's `mission.waypoint.execute`). `params` mirrors this shape:

```json
{
  "command": "start",
  "asset_sn": "<drone/gateway sn being watched>",
  "sub_asset_sn": "<camera/gimbal payload sn>",
  "task_id": "<caller-supplied correlation id>",
  "stream_url": "<rtmp/rtsp url>",
  "gimbal_tracking_enabled": false
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `command` | Yes | `"start"`, `"stop"`, `"enable_tracking"`, or `"disable_tracking"` |
| `asset_sn` | Yes | Serial number of the asset being watched (not this adapter's own SN) |
| `sub_asset_sn` | Only for gimbal aiming | Camera/gimbal payload serial number |
| `task_id` | No | Correlation id; becomes the response's `external_execution_id` if supplied |
| `stream_url` | Required for `"start"` | RTMP/RTSP URL of the video to analyze |
| `gimbal_tracking_enabled` | No, default `false` | Whether to re-aim the gimbal automatically |

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `AI_MODEL_PATH` | adapter-specific default | Path to the YOLO model file |
| `AI_FRAME_WIDTH` | `1280` | Expected video frame width (px) |
| `AI_FRAME_HEIGHT` | `720` | Expected video frame height (px) |
| `AI_CAMERA_HFOV_DEG` | `82.0` | Camera horizontal field of view (degrees) — tune to your actual camera |
| `AI_CAMERA_VFOV_DEG` | `52.0` | Camera vertical field of view (degrees) |
| `AI_MIN_CONFIDENCE` | `0.6` | Minimum detection confidence to report |
| `REMOTE_CONTROL_HOST` | `localhost` | Remote Control Service host — only used for gimbal re-aim |
| `REMOTE_CONTROL_PORT` | `8002` | Remote Control Service port |

Plus the standard `EdgeAdapterConfig` variables shared with every other Python adapter (`GRPC_HOST`/`GRPC_PORT`, `CONNECTOR_HOST`/`CONNECTOR_PORT`, `TELEMETRY_HOST`/`TELEMETRY_PORT`, `ADAPTER_SN`, `LOG_LEVEL`/`LOG_FORMAT`) — see [Configuration](edge-sdk-python-configuration.md). Set `CONNECTOR_PORT=8010` and `TELEMETRY_PORT=8003` explicitly; the library defaults don't match the platform's real ports.

Georeferencing accuracy depends entirely on accurate `AI_CAMERA_HFOV_DEG`/`AI_CAMERA_VFOV_DEG` values and good telemetry from the watched asset — treat results as approximate, not survey-grade.

---

## Running from source

```bash
uv sync
uv run --env-file .env edge-ai
```

`.env`:

```bash
GRPC_PORT=50051
CONNECTOR_HOST=localhost
CONNECTOR_PORT=8010
TELEMETRY_HOST=localhost
TELEMETRY_PORT=8003
REMOTE_CONTROL_HOST=localhost
REMOTE_CONTROL_PORT=8002
AI_MODEL_PATH=/path/to/your/model.pt
AI_CAMERA_HFOV_DEG=82.0
AI_CAMERA_VFOV_DEG=52.0
```

---

## Port

The adapter listens on `50051` for incoming gRPC commands from the platform (the standard Edge SDK default — override with `GRPC_PORT`).
