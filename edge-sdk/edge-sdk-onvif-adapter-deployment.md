# ONVIF Camera Adapter — Deployment Guide (3.0 preview)

The ONVIF adapter turns the fixed and PTZ cameras you already have at fences, masts and gates into
Zequent assets: live video in the Admin Console, PTZ commands in Applications, camera events as
alerts and event-trigger sources, and **slew-to-cue**: a radar or sensor detection points the
nearest PTZ camera that covers the target.

It speaks ONVIF Profile S/T (device, media, PTZ and events services) to each camera and relays the
camera's RTSP stream to the platform's media server. Most video management systems (VMS) expose
their cameras over ONVIF/RTSP as well, so the adapter also covers the camera side of a VMS site.

---

## Prerequisites

- Cameras that answer ONVIF (Profile S or T) and serve RTSP, reachable from the adapter
- An ONVIF user on each camera (operator rights are enough for PTZ)
- Running Zequent 3.0 platform services
- For WS-Discovery: the adapter on the cameras' network segment (host networking in a container)

---

## Cameras as assets

Each camera is an asset of type **Camera**, routed as vendor **RTMP/RTSP**. Create it in the Admin
Console with the serial number from the camera file below, or let the adapter create it once with a
pairing code (`ZQNT_CLAIM_CODE`).

The camera file describes where the camera stands and what it can see. These values are published
as the camera's telemetry (position, heading, mounting height, range, current field of view) and as
coverage on its PTZ commands, which is what the platform uses to choose a camera for a target.

```toml
[defaults]
username = "zqnt"
password_env = "ONVIF_PASSWORD"      # or password_file = "/run/secrets/onvif"

[discovery]
enabled = false                      # true: add cameras found by WS-Discovery
interval_seconds = 300

[[camera]]
sn = "CAM-GATE-NORTH"
name = "Gate north"
host = "10.0.0.21"
port = 80
latitude = 48.13710
longitude = 11.57540
ground_altitude = 520                # metres above sea level at the mast's foot
mounting_height = 8                  # metres above ground
heading = 90                         # bearing the camera faces at pan 0
horizontal_fov = 58                  # widest horizontal field of view
max_zoom = 30
pan_min = -180                       # pan range relative to heading; -180..180 = all round
pan_max = 180
tilt_min = -90
tilt_max = 20
range = 800                          # metres within which the camera can identify a target
# transcode = true                   # H.265 cameras: re-encode to H.264 for the browser
# auto_stream = true                 # on air as soon as the camera connects, see Live video
```

Passwords are never written into the file; a `password` key is refused. Cameras found by
WS-Discovery get the serial number `ONVIF-<camera serial>` and the discovery credentials; give them
a position in the file before using them for slew-to-cue.

---

## Commands

| Command | Params | Result / errors |
| --- | --- | --- |
| `camera.ptz.move` | `pan`, `tilt`, `zoom` (speed −1…1), `durationSeconds` (0.1–10, default 1) | Moves, then stops by itself |
| `camera.ptz.stop` | – | |
| `camera.ptz.goto_preset` | `preset` (name or token) | `camera.preset_not_found` names the known presets |
| `camera.ptz.look_at` | `latitude`, `longitude`, `altitude` (metres above sea level, optional), `zoom` (optional), `targetSize` (metres, default 4) | `pan`, `tilt`, `zoom`, `bearing`, `distance`; `camera.out_of_coverage` |
| `stream.start` / `stream.stop` | set by the platform | Starts/stops the live video |

`camera.ptz.look_at` turns the camera's pose and the target into pan, tilt and zoom: pan relative
to the heading, tilt from the horizon, and a zoom that frames a target of `targetSize` metres. An
omitted altitude aims at the ground the camera stands on. Targets beyond `range` or outside the pan
and tilt limits are refused with `camera.out_of_coverage`. A fixed camera offers video only; its
PTZ commands are reported as unsupported.

---

## Slew-to-cue: point a camera at a detection

1. Create an Application with one node, `camera.ptz.look_at`, and map its parameters to the
   detection: `latitude` → `$.input.latitude`, `longitude` → `$.input.longitude`,
   `altitude` → `$.input.altitude`. Add further nodes as needed (for example `stream.start`, or a
   human approval).
2. Create an event trigger of type **DETECTION** for the radar or sensor (optionally an object type
   and minimum confidence) that starts this Application, with the default target (the detecting
   asset).
3. When the radar reports a target, the run starts on the radar, and the `camera.ptz.look_at` node
   is sent to the **nearest online camera whose coverage contains the target**: within its range
   and its pan sector. A nearer camera that cannot turn far enough, or is out of range, is skipped;
   if no camera covers the target, the node fails and says which camera came closest. Later camera
   nodes of the same run go to the same camera.

Each new detection starts a new run, so the camera follows a moving target from one detection to
the next.

---

## Live video

When an operator opens the live view (or an Application runs `stream.start`), the platform sends the
media server's publish URL — `live/<sn>` on the media server, with a freshly minted publish key —
and the adapter relays the camera's RTSP stream (ONVIF `GetStreamUri`, with the camera's login) to
exactly that URL with ffmpeg, without re-encoding unless `transcode` is set. Browsers play H.264;
set `transcode = true` (or `ONVIF_TRANSCODE=true` for every camera) for H.265 cameras. The console
then plays the stream over WHEP as for any other asset, in the asset's live view and in the
execution live view.

The relay is supervised: a stream that drops is restarted with growing delays (1 s doubling to
30 s), one that cannot get going gives up after a few tries. Its state is published as telemetry
(`camera.stream`: `running`, `reconnecting`, `failed`, `stopped`) and on the `stream.start` /
`stream.stop` capabilities (`streamState`, `streamReason`), so a refused credential or an
unreachable RTSP port is visible on the asset instead of in the log alone.

### Streaming without a start from the console

A camera with `auto_stream = true` (per camera or in `[defaults]`) streams as soon as it connects
and stops reporting when it drops. Who starts it is the same choice as for the DJI adapter (see
[Live Video](../client-sdk/LIVE_VIDEO.md#automatically-when-the-camera-becomes-available)):

- **The platform.** The adapter reports that the camera can stream. With
  `LIVE_STREAM_AUTO_START_ENABLED=true` on the Admin Console API the platform answers with the
  ordinary `stream.start`, fresh publish key included, and stops the stream when the camera goes
  away. Nothing else to configure.
- **The adapter itself.** With `ONVIF_STREAM_PUBLISH_URL` (the media server, e.g.
  `rtmp://mediamtx:1936`) and `ONVIF_STREAM_PUBLISH_KEY` (a publish key an organization admin
  mints once under **Assets → Stream keys**, permission *publish*, path `live/*`) the camera
  publishes to `live/<sn>` on its own, with the key as `?user=zqnt-pub&pass=<key>` — the same path
  and form the platform uses, so the console plays it without knowing who started it. A
  `stream.start` from the platform replaces that publisher. Per camera, `publish_url` names a
  different URL and `publish_key_env` / `publish_key_file` that camera's own key (a plain
  `publish_key` in the file is refused, like `password`).

Without a key the camera publishes with a bare URL, which a media server that authenticates refuses;
the relay then reports `failed`. Without a publish URL the camera only reports that it can stream.

---

## Camera events

The adapter subscribes to each camera's events (ONVIF pull point) and publishes:

| Camera event | Alert / detection type | Severity |
| --- | --- | --- |
| Motion (motion alarm, cell motion) | `camera.motion` | Info |
| Line crossing | `camera.line_crossing` | Warning |
| Object inside a guarded field | `camera.intrusion` | Warning |
| Tamper, scene change, image too dark or blurry, signal loss | `camera.tamper` | Critical / Warning |

Each event appears as an alert in the Admin Console and as a detection of the camera with that
object type, so a **DETECTION** event trigger on the camera (object type `camera.line_crossing`,
say) starts an Application. One alert per camera and type is sent per `alert_interval_seconds`
(default 30).

---

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `ONVIF_CAMERAS_FILE` | `/etc/zqnt/onvif-cameras.toml` | The camera file |
| `ONVIF_TRANSCODE` | `false` | Re-encode every stream to H.264 |
| `ONVIF_STREAM_PUBLISH_URL` | _unset_ | The media server cameras with `auto_stream` publish to by themselves (`live/<sn>` is appended) |
| `ONVIF_STREAM_PUBLISH_KEY` | _unset_ | Their publish key (Assets → Stream keys, permission *publish*, path `live/*`) |
| `FFMPEG_BINARY` | `ffmpeg` | The relay's ffmpeg |
| `ASSET_TYPE` | _unset_ | `ASSET_TYPE_CAMERA` |
| `ASSET_VENDOR` | _unset_ | `ASSET_VENDOR_RTMP_RTSP` |
| `EDGE_ENDPOINT` | _unset_ | Where the platform reaches this adapter (`host:GRPC_PORT`) |
| `REDIS_URL` | _unset_ | The platform's Redis, for routing commands to the cameras |
| `ZQNT_EDGE_TOKEN`, `ZQNT_PLATFORM_KEYS_URL`, `ZQNT_CLAIM_CODE`, `CONNECTOR_*`, `TELEMETRY_*`, `REMOTE_CONTROL_*`, `GRPC_*`, `LOG_*` | | As for every Python adapter, see [Edge SDK Configuration](edge-sdk-python-configuration.md) |
