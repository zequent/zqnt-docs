# ONVIF Camera Adapter — Deployment Guide (3.0 preview)

The ONVIF adapter turns the fixed and PTZ cameras you already have at fences, masts and gates into
Zequent assets: live video in the Admin Console, PTZ commands in Applications, camera events as
alerts and event-trigger sources, and **slew-to-cue**: a radar or sensor detection points the
nearest free PTZ camera that covers the target and keeps it on the track.

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

## Slew-to-cue: point a camera at a detection and follow it

The platform offers two camera steps for an Application. Both pick the camera themselves: the
**nearest online PTZ camera whose coverage contains the target** (within its range and its pan
sector, as published by the adapter or corrected by an operator on the asset's Coverage tab). A
nearer camera that cannot turn far enough, or is out of range, is skipped; if no camera covers the
target, the step fails and says which camera came closest.

| Step | What it does |
|------|--------------|
| `camera.ptz.look_at` | Points the chosen camera at `latitude`/`longitude` once. Each new detection starts a new run, so a moving target is followed from one run to the next. |
| `camera.ptz.follow` | Points the nearest **free** camera at the target and keeps it on the track: the camera is re-aimed whenever the object moved more than `minMoveM` (default 5 m) or every `refreshSeconds` (default 10 s). The step keeps running until the track ends, the alarm is resolved, or someone else sends the camera a PTZ command. |

### Set it up as a site reaction

1. Create an Application with one node, `camera.ptz.follow`, and map its parameters to the
   detection: `latitude` → `$.input.latitude`, `longitude` → `$.input.longitude`,
   `trackId` → `$.input.trackId` (optional: `altitude` → `$.input.altitude`, `minMoveM`,
   `refreshSeconds`, `zoom`, `targetSize`). Release it for the site. Add further nodes as needed
   (for example `stream.start` on the camera, or a human approval).
2. Either create an event trigger of type **DETECTION** for the radar or sensor (optionally an object
   type and minimum confidence) that starts this Application, or start it from the alarm card in
   Operate: both pass the detection's position and track.
3. When the sensor reports a target, the run starts and the step points the camera. The run's
   step shows which camera was chosen and why (`CAM-01 points at track …`), every re-aim is a
   progress update, and the alarm's timeline gets a note when the camera is taken and released.

### Follow and release rules

- **Free camera.** A camera that already follows another track is not free; the next free camera
  in coverage is chosen, first come, first served. If the only cameras in coverage are busy, the
  step fails with "held by another run".
- **One follow per track.** A second run for the same track (an event trigger fires again at its
  cooldown) completes at once with "already follows track …" instead of taking a second camera.
- **The operator always wins.** A PTZ command sent to the camera by an operator (remote control,
  the command drawer) or by another run releases the follow at its next step; the step completes
  with `released = "an operator took the camera"`.
- **End of the track.** No report for 60 s (`zqnt.camera-follow.track-timeout`) ends the follow
  with `released = "track ended"`; resolving the alarm ends it with `"alarm resolved"`; cancelling
  the run releases the camera.
- **A refusal.** If the camera refuses to point (`camera.out_of_coverage`, the target left its
  range or pan/tilt limits), the step fails and the camera is released.
- Without a `trackId` the camera is pointed once and the step completes.

The step's output is `cameraSn`, `trackId`, `updates` (how often the camera was aimed), the last
`latitude`/`longitude` and `released` (why it let go). Platform defaults:
`zqnt.camera-follow.min-move-m` (5), `zqnt.camera-follow.refresh` (10s),
`zqnt.camera-follow.tick` (2s), `zqnt.camera-follow.track-timeout` (60s).

---

## Live video

When an operator opens the live view (or an Application runs `stream.start`), the platform sends the
media server's publish URL and the adapter relays the camera's RTSP stream to it with ffmpeg,
without re-encoding unless `transcode` is set. Browsers play H.264; set `transcode = true` (or
`ONVIF_TRANSCODE=true` for every camera) for H.265 cameras.

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
| `FFMPEG_BINARY` | `ffmpeg` | The relay's ffmpeg |
| `ASSET_TYPE` | _unset_ | `ASSET_TYPE_CAMERA` |
| `ASSET_VENDOR` | _unset_ | `ASSET_VENDOR_RTMP_RTSP` |
| `EDGE_ENDPOINT` | _unset_ | Where the platform reaches this adapter (`host:GRPC_PORT`) |
| `REDIS_URL` | _unset_ | The platform's Redis, for routing commands to the cameras |
| `ZQNT_EDGE_TOKEN`, `ZQNT_PLATFORM_KEYS_URL`, `ZQNT_CLAIM_CODE`, `CONNECTOR_*`, `TELEMETRY_*`, `REMOTE_CONTROL_*`, `GRPC_*`, `LOG_*` | | As for every Python adapter, see [Edge SDK Configuration](edge-sdk-python-configuration.md) |
