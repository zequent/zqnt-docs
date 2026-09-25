# Live Video Streams

This page covers live video on the **1.3.x line**: how a stream gets on air, how to start and stop
one from your own application, and how to play it back.

> **The platform does not carry your video.** It is the control plane only. The device pushes RTMP
> straight to a media server, and your player pulls from that same media server. No video frame ever
> passes through a Zequent service, so stream quality and latency are a property of your media
> server and the device's uplink, not of the platform.

## Which assets can stream

| Adapter (released 1.3.x) | Live video |
| --- | --- |
| **DJI** `1.3.1` | **Yes** — `startLiveStream` / `stopLiveStream`, plus lens, zoom and split-screen |
| **MAVLink** `1.3.0` | No |
| **Betaflight** `1.3.0` | No |
| **RNS** `1.3.0` | No |
| **SAPIENT** `1.3.0` | No |
| **Simulator** `1.3.x` | No |

On 1.3.x, live video is a **DJI-only** capability. The Edge SDK declares `start_live_stream` /
`stop_live_stream` for every adapter, but only the DJI adapter implements them — the others inherit
the default and answer *not supported*. If you call the API against a MAVLink aircraft you get that
answer back, not a stream.

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
becomes available, the adapter starts the stream **on its own**, and stops it again when the camera
goes away. Nobody has to press anything, and no API call is involved.

This happens only when the aircraft's record carries both:

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
  "streamUrl": "https://media.example.com/live/1581F5BMD227T00A1234/DRONE",
  "success": true,
  "message": "Livestream started successfully",
  "timestamp": "2026-09-22T09:14:02Z"
}
```

`GET /api/admin-console/streams/{sn}` returns the same `streamUrl` alongside `live`, `videoId`,
`assetType` and `startedAt`. It is the right call to make when a page loads, because a stream may
already be running — started automatically, or by another operator.

Access control on these endpoints is a property of your deployment (typically the reverse proxy in
front of the Admin Console API), not of the API itself.

## Option B — the Client SDK

One level lower: the same call the Admin Console API makes, straight to the Live Data service. Use
it when your application already speaks to the platform through the SDK and you do not want to
depend on the Admin Console API.

**What you give up:** the SDK does not resolve anything for you. Your application supplies the
`videoId` and the RTMP push URL itself, and derives the playback URL itself. Everything in
[Identifying the camera](#identifying-the-camera-videoid) and
[Where the push URL comes from](#where-the-push-url-comes-from) becomes your responsibility.

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
                .streamServer("rtmp://media.example.com/live/1581F5BMD227T00A1234/DRONE")
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
            stream_server="rtmp://media.example.com/live/1581F5BMD227T00A1234/DRONE",
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
your own TLS and retry policy:

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
`live_stream_start`); on the 1.3.x line the platform never fills it, so it is always absent.

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

On the **on-demand** path through the Admin Console API, the push URL is built from the deployment's
configured base URL:

```
${LIVE_STREAM_SERVER_URL}/{sn}/{DRONE|DJI_DOCK}
```

and the playback URL is that same path re-pointed at `${LIVE_WHEP_BASE_URL}`, preserving any query
string.

| Variable | Set on | Meaning |
| --- | --- | --- |
| `LIVE_STREAM_SERVER_URL` | Admin Console API | RTMP base the device publishes to, e.g. `rtmp://media.example.com/live` |
| `LIVE_WHEP_BASE_URL` | Admin Console API | HTTP base your player pulls from, e.g. `https://media.example.com` |

> Because the two paths derive the URL differently, the same aircraft can publish to a different URL
> depending on whether the stream was started automatically or on demand. If you use both, keep
> `liveStreamPushUrl` consistent with what `LIVE_STREAM_SERVER_URL` produces for that serial.

Live video is a licensed feature. Without a valid licence lease covering live streaming, the start
call is refused before it reaches the device.

## Troubleshooting

**Start reports success but no video arrives.** The platform keeps a record of which streams are
live, and a start for a stream it already believes is live returns success **without contacting the
device** — the message says so (*"Stream for SN is already live"*). If the device is in fact not
streaming, stop the stream first, then start it again.

**The call times out although the stream comes up.** Both the client SDK and the adapter allow about
30 seconds — the SDK for the whole call, the adapter for the dock's reply. A slow dock can therefore
surface as a client-side timeout on a command that succeeded. Raise the SDK's request timeout if you
see this; do not retry blindly, or you will start and immediately re-start the same camera.

**The device rejects the start.** Nearly always the `videoId` or the push URL. Check the `videoId`
against the aircraft's real serial and payload index, and confirm the media server accepts a publish
at that path.

**Nothing at all happens for an aircraft that is not DJI.** Expected — see
[Which assets can stream](#which-assets-can-stream).

**The stream stops by itself.** If the dock reports the camera as no longer available, the adapter
stops the stream deliberately. Landing, powering down or switching payloads all produce this.

## See also

- [Java Client SDK Quickstart](QUICKSTART.md) · [Python](QUICKSTART_PYTHON.md) · [Go](QUICKSTART_GO.md)
- [Remote Control](REMOTE_CONTROL.md) — camera and gimbal commands, including split-screen
- [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) — why an aircraft camera is addressed
  by the aircraft serial and not the dock's
- [DJI Adapter Deployment](../edge-sdk/edge-sdk-dji-adapter-deployment.md)
