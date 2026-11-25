# AI Edge Adapter — Deployment Guide

The AI Adapter pulls a live RTMP/RTSP video stream, runs YOLO object detection/tracking on it, georeferences each detection against the target camera's live telemetry (turning a bounding box into an approximate lat/lon), publishes the results to the platform, and can optionally re-aim the target's gimbal at whatever it's currently tracking.

Unlike every other adapter, this one doesn't drive the hardware it watches — it registers as its **own** logical asset (`ASSET_TYPE_SENSOR`) and observes a *different* asset's telemetry and video as a client, using the same Live Data streaming and Remote Control gRPC contracts every other adapter and the Client SDK use.

The adapter is built on the standard `EdgeAdapterRuntime`/`EdgeAdapterConfig` pattern (the same one every other Python adapter uses). The published image is `ghcr.io/zequent/zqnt-adapter-ai:2.0.0`; you can also run it from source.

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

## Vision models (3.0)

On 3.0 the adapter runs **versioned models from a model catalog**, offers one command per task
(detect objects, read licence plates, detect drones), keeps the GPU within a budget, and can run a
second model on what the first one found, such as reading the text of a detected licence plate.

### The model catalog

The catalog is a `catalog.json` in the model directory (`AI_MODEL_DIR`, default `models/`). Each
entry describes one model file:

```json
{"models": [
  {"name": "plates", "version": "1.2.0", "task": "read_plate", "file": "plates-1.2.0.onnx",
   "sha256": "<64 hex characters>", "classes": ["licence_plate"], "licence": "Apache-2.0",
   "vram_mb": 400, "ms_per_frame": 8,
   "second_stage": {"model": "plate-text", "classes": ["licence_plate"], "max_crops": 4}},
  {"name": "plate-text", "version": "2.0.0", "kind": "text", "attribute": "plate",
   "file": "plate-text-2.0.0.onnx", "sha256": "<64 hex characters>", "licence": "MIT",
   "alphabet": "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ_", "pad": "_"}
]}
```

| Field | Meaning |
| --- | --- |
| `name`, `version` | The model's name and version. Every detection names the model that found it as `name@version` |
| `task` | What the model is for: `detect` (default), `read_plate`, `detect_uav` or a task name of your own |
| `kind` | `detector` (default), or `text` / `classifier` for a model that only runs as another model's second stage |
| `file` | The ONNX file, next to `catalog.json` |
| `sha256` | The file's checksum. A file that does not match is not offered |
| `classes` | The classes the model reports |
| `licence` | The model's licence, shown with the model |
| `vram_mb`, `ms_per_frame` | GPU memory and time per frame, used for the GPU budget below |
| `second_stage` | A model to run on this detector's findings (see below) |
| `attribute` | For a `text` or `classifier` model: the name of what it adds to a detection, e.g. `plate` |
| `alphabet`, `pad` | For a `text` model: the characters it reads, and the character that fills unused positions |

An `*.onnx` file in the directory without a catalog entry still works: it is a `detect` model named
after its file, version `local`, licence unknown. Your existing models keep working without a
catalog.

### Distributing models to sites

Point the adapter at a catalog in object storage, on a web server or on disk, with the Technical
Config setting `ai.catalog.url` (for one adapter or for every AI adapter) or `AI_MODEL_CATALOG_URL`,
for example `s3://models/catalog.json`. The model files lie next to the catalog. An `s3://` catalog
is read with `AI_MODEL_S3_ENDPOINT`, `AI_MODEL_S3_ACCESS_KEY` and `AI_MODEL_S3_SECRET_KEY` (by default
the adapter's `AI_S3_*` credentials).

Every `AI_MODEL_CATALOG_POLL_S` seconds (default 300) the adapter downloads what changed, checks
every file against its `sha256`, swaps it in and offers the new models, without a restart. Models
that an earlier download brought and the catalog no longer lists are removed. A distributed catalog
must give a checksum for every model: a catalog with an entry without one is refused as a whole,
and the site keeps the models it has. `detection.status` shows the catalog, any entry that could not
be used and why.

To roll a site back or move it to another set of models, change `ai.catalog.url`.

### One command per task

Each task in the catalog has its own command in the Skill editor, and its `model` parameter offers
that task's models as a list to choose from:

| Command | Task | Shown as |
| --- | --- | --- |
| `detection.start` | `detect` | Start AI detection |
| `detection.read_plate` | `read_plate` | Read licence plates |
| `detection.detect_uav` | `detect_uav` | Detect drones |
| `detection.<task>` | any other task | the task's name |

A task's command appears only while the site has a model for it, and goes away when the catalog no
longer has one. A model of another task is refused. When a node names no model, the adapter uses
the Technical Config setting `ai.model.<task>` (for example `ai.model.read_plate`), else the task's
first model. The task commands take the same parameters as `detection.start`.

### Two-stage models: reading what was detected

A detector's `second_stage` names a `text` or `classifier` model that runs on the image crops of
its detections: the largest first, at most `max_crops` per frame (default 4), until each object
has a reading. A later, more confident reading replaces an earlier one. A `text` model reads text,
such as a plate number; a `classifier` picks one label, such as a drone type.

The result becomes an **attribute** of the detection, with its own confidence and the model that
produced it, for example `plate = M AB 123` (93 %) from `plate-text@2.0.0`. The platform stores it
with the detection, `detection.wait` returns it under `attributes`, and the Admin Console shows it
in the track's details on the **Detections** page (`Plate: M AB 123 (93%)`), together with the model
that saw the object. See [Detection attributes](edge-sdk-python-adapter-3.0.md#detection-attributes)
for reporting attributes from your own adapter.

### Model formats

Models are ONNX files. The adapter understands the output of YOLOv5/7, YOLOv8/11/12, end-to-end
models with built-in NMS, and DETR-style detectors (RT-DETR, RF-DETR, D-FINE). Text models with
fixed character slots (as fast-plate-ocr exports them) and CTC text models (as PaddleOCR exports
them) are understood. Class names come from the model's metadata or from a `<model>.names` file
next to it, one name per line.

### GPU budget: refuse or queue

Every detection run reserves part of the GPU: its frames per second × milliseconds per frame (more
when tiling, plus its second stage) and the memory of its models. Runs that analyse the same video
with the same model and settings share one analysis and pay once. The milliseconds per frame come
from the catalog entry, else from what the adapter measures.

| Budget | Default | Change with |
| --- | --- | --- |
| Inference time | 900 ms per second | `AI_GPU_COMPUTE_MS_PER_S` or `ai.gpu.compute_ms_per_s` |
| GPU memory | 90 % of the GPU's memory when inference runs on a GPU; no limit on the CPU | `AI_GPU_VRAM_MB` or `ai.gpu.vram_mb` |

A start that does not fit is **refused** with the numbers (reason `detection.over_budget`), so the
runs already going, which may be the ones that matter for safety, stay on time. With
`when_busy: queue` the start is accepted instead, answers `queued: true`, and starts as soon as
another run stops and there is room, oldest first. `detection.stop` withdraws a queued run.
`detection.status` shows the budget, what each run uses and the queue.

### Many cameras on one GPU

On a GPU each model has one inference worker. The frames of all streams that use the model queue
there instead of calling the GPU at the same time. A model exported with a **dynamic batch size** gets the queued frames of
several streams, and the tiles of a tiled frame, in one call, up to `AI_MAX_BATCH` (default 8)
frames; a model with a fixed batch of 1 runs them one after the other.

When the GPU falls behind, a stream's older waiting frame is dropped for its newer one, so
detection stays current instead of lagging further and further behind.

> **Tip.** Export your models with a dynamic batch dimension (for Ultralytics models,
> `dynamic=True`). The GPU then spends less time per frame and has more room for further cameras.

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `AI_MODEL_PATH` | `yolov12_custom.onnx` | Path to the YOLO model file |
| `AI_FRAME_WIDTH` | `1280` | Expected video frame width (px) |
| `AI_FRAME_HEIGHT` | `720` | Expected video frame height (px) |
| `AI_CAMERA_HFOV_DEG` | `82.0` | Camera horizontal field of view (degrees) — tune to your actual camera |
| `AI_CAMERA_VFOV_DEG` | `52.0` | Camera vertical field of view (degrees) |
| `AI_MIN_CONFIDENCE` | `0.6` | Minimum detection confidence to report |
| `ZQNT_CLAIM_CODE` | _unset_ | A one-time pairing code from the console's **Pair device** dialog. Used only when the platform does not already know this service's serial number: the code decides which organization the resulting asset belongs to, and that cannot be changed afterwards. Leave it unset once the asset exists — the code is single-use. Without it, an unknown serial simply has no asset, and this service creates nothing. |
| `ZQNT_EDGE_TOKEN` | _unset_ | The adapter's **edge credential**, sent on every call to the platform. Issued in the Admin Console under **Manage → Access & Integrations → Credentials** (kind *Edge adapter*); without it the platform refuses the calls |
| `ZQNT_PLATFORM_PUBLIC_KEY` | _unset_ | The platform's public key (alias `SERVICE_AUTH_PUBLIC_KEY`). The adapter refuses every command the platform did not sign with it; without it, every command is refused |
| `ZQNT_EDGE_AUTH_DISABLED` | `false` | `true` accepts commands from anyone who can reach the port — local stacks only |
| `REMOTE_CONTROL_HOST` | `localhost` | Remote Control Service host — only used for gimbal re-aim |
| `REMOTE_CONTROL_PORT` | `8002` | Remote Control Service port |
| `AI_MODEL_DIR` | `models` | 3.0: directory of the model catalog and the `*.onnx` files |
| `AI_MODEL_CATALOG_URL` | _unset_ | 3.0: catalog to distribute to this adapter (`s3://`, `https://` or a path); `ai.catalog.url` overrides it |
| `AI_MODEL_CATALOG_POLL_S` | `300` | 3.0: how often the catalog is checked for changes (seconds) |
| `AI_MODEL_S3_ENDPOINT`, `AI_MODEL_S3_ACCESS_KEY`, `AI_MODEL_S3_SECRET_KEY` | the `AI_S3_*` values | 3.0: read credentials for an `s3://` catalog |
| `AI_GPU_COMPUTE_MS_PER_S` | `900` | 3.0: inference time budget in milliseconds per second; `ai.gpu.compute_ms_per_s` overrides it |
| `AI_GPU_VRAM_MB` | 90 % of the GPU | 3.0: GPU memory budget; `ai.gpu.vram_mb` overrides it |
| `AI_MAX_BATCH` | `8` | 3.0: frames per GPU call for a model with a dynamic batch size |

Plus the standard `EdgeAdapterConfig` variables shared with every other Python adapter (`GRPC_HOST`/`GRPC_PORT`, `CONNECTOR_HOST`/`CONNECTOR_PORT`, `TELEMETRY_HOST`/`TELEMETRY_PORT`, `ADAPTER_SN`, `LOG_LEVEL`/`LOG_FORMAT`) — see [Configuration](edge-sdk-python-configuration.md). Set `CONNECTOR_PORT=8010` and `TELEMETRY_PORT=8003` explicitly; the library defaults don't match the platform's real ports.

Georeferencing accuracy depends entirely on accurate `AI_CAMERA_HFOV_DEG`/`AI_CAMERA_VFOV_DEG` values and good telemetry from the watched asset — treat results as approximate, not survey-grade.

---

## Running the published image

```bash
docker run --env-file .env -p 50051:50051 ghcr.io/zequent/zqnt-adapter-ai:2.0.0
```

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
