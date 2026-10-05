# Live Video Streams

This page covers live video: how a stream gets on air, how to start and stop one from your own
application, how to play it back, and how to lock the media server down with stream keys.

> **The platform does not carry your video.** It is the control plane only. The device pushes RTMP
> straight to a media server, and your player pulls from that same media server. No video frame ever
> passes through a Zequent service, so stream quality and latency are a property of your media
> server and the device's uplink, not of the platform.

## Which assets can stream

| Adapter (`2.0.0`) | Live video |
| --- | --- |
| **DJI** | **Yes** — `startLiveStream` / `stopLiveStream`, plus lens, zoom and split-screen |
| **Simulator** | **Yes** — a test pattern published with `ffmpeg` (see below) |
| **MAVLink** | No |
| **Betaflight** | No |
| **RNS** | No |
| **SAPIENT** | No |
| **AI** | No — it reads streams from the media server, it does not publish one |

The Edge SDK declares `start_live_stream` / `stop_live_stream` for every adapter, but only the DJI
adapter and the simulator implement them — the others inherit the default and answer *not
supported*. If you call the API against a MAVLink aircraft you get that answer back, not a stream.

The **simulator** publishes colour bars with the device's serial and telemetry burnt in, to exactly
the URL the platform sent, credential included. It needs `ffmpeg` on the simulator's host (or
`SIM_FFMPEG` pointing at one). `SIM_STREAM_VIDEO` publishes a video file in a loop instead, and
`SIM_STREAM_VIDEO_FOR` limits that video to some serials (comma-separated globs, e.g. `SIM-SXF-*`).

For DJI, the stream belongs to the **aircraft camera**. The dock only acts as the gateway that
carries the command to the aircraft; you do not start a stream "on a dock".

## How a stream gets on air

```
  Zequent platform  ──── start command ────►  DJI dock  ────►  aircraft
   (control only)                                                  │
                                                                   │ RTMP push
                                                                   ▼
   your player  ◄──── WebRTC / WHEP ────────────────────────  media server
```

1. Something asks the platform to start a stream for a given camera.
2. The platform tells the aircraft, through the dock, which URL to push to.
3. The aircraft pushes RTMP to your media server.
4. Your player (the Admin Console, or your own page) pulls the same stream back over WebRTC/WHEP.

## Two ways a stream starts

### Automatically, when the camera becomes available

The DJI adapter watches what the dock reports about its camera capacity. When an aircraft's camera
becomes available, the stream starts **on its own**, and stops again when the camera goes away.
Nobody has to press anything. Who sends the start command is a deployment choice:

| Mode | Set | What happens |
| --- | --- | --- |
| **Adapter** (default) | nothing | The DJI adapter starts the stream itself, with the push URL stored on the aircraft's record |
| **Platform** | `livestream.autostart.mode=platform` on the DJI adapter **and** `LIVE_STREAM_AUTO_START_ENABLED=true` on the Admin Console API | The adapter only reports that the camera can stream; the platform answers with the ordinary start — the same one as the console button, with a freshly minted push URL |

Set both or neither: with only one of them, either nobody starts the stream or the adapter keeps
starting it. Use **platform** mode when the media server authenticates (see
[Stream keys](#authenticated-media-server-stream-keys)) — the adapter cannot mint a credential, so in
adapter mode it publishes with whatever was stored on the record.

In adapter mode the start happens only when the aircraft's record carries both:

- `streamUrlPredefined = true`
- a non-empty `liveStreamPushUrl` — the RTMP URL that aircraft should push to

If either is missing the adapter skips the auto-start silently; it is a normal condition, not an
error. There is a short delay (about 5 seconds) between the camera becoming available and the start
command being sent, so the aircraft has settled before it is asked to publish.

### On demand, from the console or your application

The rest of this page is about this path: an explicit start for a named camera.

## Identifying the camera: `videoId`

Every start and stop call names a camera with a `videoId` in DJI Cloud API form:

```
{deviceSn}/{payloadIndex}/{videoType}
```

| Part | Value |
| --- | --- |
| `deviceSn` | the **aircraft** serial number for an aircraft camera; the dock serial for a dock camera |
| `payloadIndex` | the payload's `externalId` on that record — e.g. `99-0-0` for an aircraft camera, `165-0-7` for a dock camera |
| `videoType` | `normal-0` |

So a typical aircraft camera is `1581F5BMD227T00A1234/99-0-0/normal-0`.

The **first segment decides which record the platform tracks the stream against**. If you send a
`videoId` whose first segment is not the camera's own device serial, the stream may well play while
the platform reports the wrong asset as live. Use the real serial.

## Option A — the Admin Console API

The simplest route, and the one the Admin Console itself uses. It works out the `videoId` and the
push URL for you, and hands back a playback URL you can put straight into a player.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/admin-console/streams/{sn}` | Current state: is it live, which camera, playback URL |
| `POST` | `/api/admin-console/streams/{sn}/start` | Start the stream |
| `POST` | `/api/admin-console/streams/{sn}/stop` | Stop the stream |
| `POST` | `/api/admin-console/streams/{sn}/lens` | Switch the active lens |
| `POST` | `/api/admin-console/streams/{sn}/zoom` | Change zoom |

Start a stream, letting the platform choose the camera:

```bash
curl -X POST http://admin-console:8005/api/admin-console/streams/1581F5BMD227T00A1234/start \
     -H 'Content-Type: application/json' \
     -d '{}'
```

`videoId` and `streamType` are both optional in the body. Supply `videoId` only when you want a
camera other than the default one; supply `streamType` (`RTMP`, `RTSP`, `WEBRTC`) only when you want
to override what the platform picks.

The response describes the command, and for a successful start it carries the playback URL:

```json
{
  "sn": "1581F5BMD227T00A1234",
  "tid": "0b2c1e8a-...",
  "videoId": "1581F5BMD227T00A1234/99-0-0/normal-0",
  "streamUrl": "https://media.example.com/live/1581F5BMD227T00A1234/drone",
  "success": true,
  "message": "Livestream started successfully",
  "timestamp": "2026-09-22T09:14:02Z"
}
```

`GET /api/admin-console/streams/{sn}` returns the same `streamUrl` alongside `live`, `videoId`,
`assetType` and `startedAt`. It is the right call to make when a page loads, because a stream may
already be running — started automatically, or by another operator.

Every endpoint needs a signed-in console user. Reading the state is open to any of them; start,
stop, lens and zoom need the **operator** role.

When the Admin Console API knows the media server's API (`LIVE_STREAM_MEDIA_SERVER_API_URL`),
`live` reflects whether video is really arriving there, not just whether the device accepted the
start command.

## Option B — the Client SDK

One level lower: the same call the Admin Console API makes, straight to the Live Data service. Use
it when your application already speaks to the platform through the SDK and you do not want to
depend on the Admin Console API.

**What you give up:** the SDK does not resolve anything for you. Your application supplies the
`videoId` and the RTMP push URL itself, and derives the playback URL itself. Everything in
[Identifying the camera](#identifying-the-camera-videoid) and
[Where the push URL comes from](#where-the-push-url-comes-from) becomes your responsibility — and,
when the media server authenticates, so does the credential: the SDK cannot mint one, so put a
[stream key](#stream-keys-for-devices-the-platform-cannot-start) into the push URL.

### Java

```java
import com.zqnt.sdk.client.livedata.domains.LiveDataStartLiveStreamRequest;
import com.zqnt.sdk.client.livedata.domains.LiveDataStopLiveStreamRequest;
import com.zqnt.utils.common.proto.AssetTypeEnum;
import com.zqnt.utils.common.proto.LiveStreamTypeEnum;

var response = client.liveData().startLiveStream(
        LiveDataStartLiveStreamRequest.builder()
                .sn("1581F5BMD227T00A1234")
                .videoId("1581F5BMD227T00A1234/99-0-0/normal-0")
                .streamServer("rtmp://media.example.com/live/1581F5BMD227T00A1234/drone")
                .streamType(LiveStreamTypeEnum.LIVE_STREAM_TYPE_RTMP)
                .assetType(AssetTypeEnum.ASSET_TYPE_AIRCRAFT)
                .build())
        .get();

if (response.getHasErrors()) {
    log.warn("start failed: {}", response.getError().getErrorMessage());
}

// later
client.liveData().stopLiveStream(
        LiveDataStopLiveStreamRequest.builder()
                .sn("1581F5BMD227T00A1234")
                .videoId("1581F5BMD227T00A1234/99-0-0/normal-0")
                .build())
        .get();
```

The `tid` you set on the request is **not** sent — the Java SDK always generates a fresh one. Read
the one that was actually used from `response.getMeta().getTid()`.

### Python

```python
from client_sdk import (
    ZequentClient,
    LiveDataStartLiveStreamRequest,
    LiveDataStopLiveStreamRequest,
)

async with ZequentClient.from_env() as client:
    result = await client.live_data.start_live_stream(
        LiveDataStartLiveStreamRequest(
            sn="1581F5BMD227T00A1234",
            video_id="1581F5BMD227T00A1234/99-0-0/normal-0",
            stream_server="rtmp://media.example.com/live/1581F5BMD227T00A1234/drone",
        )
    )
    if not result.success:
        print("start failed:", result.error_message)

    await client.live_data.stop_live_stream(
        LiveDataStopLiveStreamRequest(
            sn="1581F5BMD227T00A1234",
            video_id="1581F5BMD227T00A1234/99-0-0/normal-0",
        )
    )
```

`stream_type` defaults to `RTMP` and `asset_type` to `AIRCRAFT`. Unlike Java, the Python SDK sends
the `tid` you set, and rejects a blank `video_id` or `stream_server` before the call leaves your
process.

### Go

The Go SDK is a thin wrapper — you build the protobuf request yourself and dial the connection with
your own TLS and retry policy (`livedata.New(conn)`, dialled as in the [Go quickstart](QUICKSTART_GO.md)):

```go
resp, err := liveDataClient.StartLiveStream(ctx, &devicecontrol.LiveStreamStartCommandRequest{
    Base:    &base.RequestBase{Sn: sn, Tid: uuid.NewString()},
    Request: &devicecontrol.LiveStreamStartCommandPayload{
        VideoId:      videoID,
        StreamServer: pushURL,
        StreamType:   common.LiveStreamTypeEnum_LIVE_STREAM_TYPE_RTMP,
        AssetType:    common.AssetTypeEnum_ASSET_TYPE_AIRCRAFT,
    },
})
```

### The start response does not contain a playback URL

Over the SDK, a successful start returns success, `tid`, `sn` and a message — **not** a playback
URL. The response type has a field that looks like it should carry one (`liveStreamStartResponse` /
`live_stream_start`); the platform never fills it, so it is always absent.

Build the playback URL yourself from the RTMP URL you supplied — same path, pointed at your media
server's WHEP endpoint, query string preserved — or call
`GET /api/admin-console/streams/{sn}`, which returns the finished URL.

## Camera control

Available alongside the stream, for DJI:

| Control | Admin Console API | Client SDK |
| --- | --- | --- |
| Switch lens | `POST /streams/{sn}/lens` with `{"lens": "wide"}` | `liveData().changeCameraLens(...)` |
| Zoom | `POST /streams/{sn}/zoom` with `{"lens": "zoom", "zoom": 5}` | `liveData().changeCameraZoom(...)` |
| Split-screen | — | `remoteControl().liveStreamSplitScreen(...)` |

Valid `lens` values are `wide`, `zoom` and `ir`.

**Split-screen needs a running stream on the thermal lens.** The command switches the camera to `ir`
first and then enables the split view; with no stream running it answers that there is no active
video to split, which reads like a failure but is the expected answer.

## Where the push URL comes from

On the **automatic** path, the aircraft pushes to the `liveStreamPushUrl` stored on its own record.

On the **on-demand** path through the Admin Console API (and in platform auto-start mode), the base
is the asset's own `liveStreamPushUrl` when it has one, and `LIVE_STREAM_SERVER_URL` otherwise. The
platform then adds the serial and, for a DJI dock, which of its two cameras:

```
{base}/{sn}/drone       aircraft camera
{base}/{sn}/dji_dock    dock camera
{base}/{sn}             any other asset — it has one stream
```

A base that already contains the serial is used as it is. Media-server paths are case-sensitive:
the segments are lower case.

The playback URL is that same path re-pointed at `${LIVE_WHEP_BASE_URL}`, preserving any query
string.

| Variable | Set on | Meaning |
| --- | --- | --- |
| `LIVE_STREAM_SERVER_URL` | Admin Console API | RTMP base the device publishes to, e.g. `rtmp://media.example.com/live` |
| `LIVE_WHEP_BASE_URL` | Admin Console API | HTTP base your player pulls from, e.g. `https://media.example.com` |
| `LIVE_STREAM_MEDIA_SERVER_API_URL` | Admin Console API | Optional. The media server's API, used to check that a stream is really live |
| `LIVE_STREAM_AUTO_START_ENABLED` | Admin Console API | `true` for platform auto-start mode (default `false`) |
| `LIVE_STREAM_AUTH_ENABLED` | Admin Console API | `true` when the media server authenticates (default `false`) — see below |
| `LIVE_STREAM_AUTH_HOOK_SECRET` | Admin Console API | Secret the media server must send to the authentication hook |
| `LIVE_STREAM_KEYS_PEPPER` | Admin Console API | Secret mixed into stored stream-key hashes |

## Authenticated media server (stream keys)

By default the media server takes anything that connects. To require a credential, configure the
media server to ask the platform (MediaMTX: `authMethod: http`, with `authHTTPAddress` pointing at
`http://admin-console:8005/api/admin-console/streams/auth?secret=<hook secret>`) and set
`LIVE_STREAM_AUTH_ENABLED=true` on the Admin Console API. Switch both on together: with only the
platform side on, URLs carry credentials nobody checks; with only the media server side on, every
stream is refused.

### Keys the platform mints itself

Every start through the Admin Console API mints a short-lived credential for that one stream path:

- a **publish** key, good for 10 minutes, put into the push URL the device receives;
- a **read** key, good for 2 minutes, put into the playback URL a viewer receives — a fresh one on
  every state call while the stream is live.

For RTMP, RTSP and SRT the key travels in the query (`?user=zqnt-pub&pass=<key>`); for an HTTP
playback URL it travels as `zqnt-read:<key>@host`. The publish key never comes back to whoever
started the stream.

### Stream keys for devices the platform cannot start

A camera started from its own web interface, a third-party encoder, or your own application
starting streams through the Client SDK needs a longer-lived key. An organization admin cuts one in
the Admin Console in the **Stream keys** panel on the Assets page (or with
`POST /api/admin-console/streams/keys`):

| Field | Meaning |
| --- | --- |
| `path` | The media-server path the key is for, e.g. `live/1581F5BMD227T00A1234/drone` |
| `permission` | `publish` or `read` |
| `assetSn`, `label` | What it is for, so it can be found later |
| `ttlDays` | Lifetime: 30 days by default, at most 90 |

The key is shown **once**, in the response; only its hash is stored. Present it as the password, or
as `?key=<key>` on the URL. `GET /api/admin-console/streams/keys` lists the organization's keys with
their last use, and `DELETE /api/admin-console/streams/keys/{id}` revokes one. A connection already
open with a revoked key keeps running until it reconnects.

Live video is a licensed feature. Without a valid licence lease covering live streaming, the start
call is refused before it reaches the device.

## Troubleshooting

**Start reports success but no video arrives.** The platform keeps a record of which streams are
live, and a start for a stream it already believes is live returns success **without contacting the
device** — the message says so (*"Stream for SN is already live"*). If the device is in fact not
streaming, stop the stream first, then start it again.

**The call times out although the stream comes up.** The Client SDK allows 30 seconds by default
for the whole call. A slow dock can therefore surface as a client-side timeout on a command that
succeeded. Raise the SDK's request timeout if you see this; do not retry blindly, or you will start
and immediately re-start the same camera.

**The start succeeds but the player stays black, with authentication on.** The media server refused
the credential. Check that the device was given the URL unchanged, that the media server sends the
hook secret, and — in adapter auto-start mode — that you have switched to platform mode.

**The device rejects the start.** Nearly always the `videoId` or the push URL. Check the `videoId`
against the aircraft's real serial and payload index, and confirm the media server accepts a publish
at that path.

**Nothing at all happens for an asset that is not DJI or the simulator.** Expected — see
[Which assets can stream](#which-assets-can-stream).

**The stream stops by itself.** If the dock reports the camera as no longer available, the adapter
stops the stream deliberately. Landing, powering down or switching payloads all produce this.

## See also

- [Java Client SDK Quickstart](QUICKSTART.md) · [Python](QUICKSTART_PYTHON.md) · [Go](QUICKSTART_GO.md)
- [Remote Control](REMOTE_CONTROL.md) — camera and gimbal commands, including split-screen
- [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) — why an aircraft camera is addressed
  by the aircraft serial and not the dock's
- [DJI Adapter Deployment](../edge-sdk/edge-sdk-dji-adapter-deployment.md)
