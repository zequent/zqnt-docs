# Forwarding live telemetry (3.0 preview)

The **Zequent Telemetry** source sends the live telemetry of your assets out of the platform, to any
sink Integration Hub supports: MQTT, Kafka, OPC UA, WebSocket or an HTTP API. Use it to feed a
control room, a data lake or a third-party dashboard with positions, battery levels and device values
as they arrive, without writing code against the Client SDK.

This page describes a preview; details can change until 3.0 is released.

## Before you start

- Integration Hub must be connected to the platform (`ZQNT_PLATFORM_ENABLED=true` and an integration
  credential in `ZQNT_EDGE_TOKEN`, see [Zequent Platform
  Integration](integration-hub.md#backend----zequent-platform-integration-optional)). It reads the
  telemetry from the Live Data Service at `ZQNT_LIVE_DATA_HOST` / `ZQNT_LIVE_DATA_PORT` (default
  `live-data-service` / `8003`).
- A telemetry connector belongs to one organization and only ever sends that organization's assets.
  Create it signed in as a user of that organization; a system admin picks the organization in the
  form.

## Settings

In the console, open **Manage → Integrations → Connectors**, create a connector with role **Source**
and type **Zequent Telemetry (live asset data)**. The form has three settings; the same values appear
in the connector's JSON config.

| Setting | Config key | What it does |
| --- | --- | --- |
| Assets | `assetSns` | The serial numbers of the assets to forward. Leave it empty to forward every asset of your organization, including assets added later. Selecting a dock includes its aircraft. |
| Fields | `fields` | The values to send. Leave it empty to send all of them. |
| Minimum interval | `minIntervalMs` | At most one message per asset in this many milliseconds. Default `1000`, lowest `100`. |

The fields you can pick:

| Field | Content |
| --- | --- |
| `position` | `latitude`, `longitude` and `altitude` (metres above sea level). Pick `position.latitude`, `position.longitude` or `position.altitude` for one of them only. |
| `relativeAltitude` | Metres above the takeoff point |
| `headingDegrees` | Degrees clockwise from true north |
| `horizontalSpeed` | m/s |
| `verticalSpeed` | m/s, positive up |
| `batteryPercent` | Battery level in percent |
| `details` | Every device-specific value the asset reports, such as the wind speed |
| `details.<key>` | One device-specific value, for example `details.wind.speed` |

`assetSn` and `observedAt` are always sent, whatever you pick. A value the asset did not report is
left out of the message rather than sent as zero.

## What a message looks like

Each message is one sample of one asset, here with no fields selected:

```json
{
  "assetSn": "DRONE-7",
  "observedAt": "2026-10-21T19:53:17.574Z",
  "position": { "latitude": 52.5232, "longitude": 13.4045, "altitude": 74 },
  "relativeAltitude": 40,
  "headingDegrees": 57.4,
  "batteryPercent": 76,
  "details": { "wind.speed": 3.4 }
}
```

A bridge without a mapping passes this through unchanged. A mapping sends only the fields it maps,
so map `assetSn` too if the sink needs it, for example in an MQTT topic.

## Example: selected drones to MQTT, once per second

A control room wants the position and battery of two drones on its MQTT broker, one message per
drone per second, on a topic per drone.

1. **Source.** Create a connector named `fleet-telemetry`, role **Source**, type **Zequent
   Telemetry**. Under **Assets** tick the two drones, under **Fields** pick **Position** and
   **Battery**, and leave **Minimum interval** at `1000`. The config reads:

   ```json
   {
     "assetSns": ["DRONE-7", "DRONE-9"],
     "fields": ["position", "batteryPercent"],
     "minIntervalMs": 1000
   }
   ```

2. **Sink.** Create a connector named `control-room-mqtt`, role **Sink**, type **MQTT**, with the
   broker address and the topic `zequent/telemetry/{assetSn}`. `{assetSn}` is replaced with each
   message's serial number, so `DRONE-7` publishes to `zequent/telemetry/DRONE-7`. Store the broker's
   username and password as a secret under **Manage → Access & Integrations → Secrets → Integration
   Hub** and select it in the connector form's **Secret** field.

3. **Bridge.** Under **Bridges**, create a bridge from `fleet-telemetry` to `control-room-mqtt`,
   leave **Mapping** at "No mapping — pass the source payload through" and start it. A telemetry
   source pushes its messages, so the bridge has no poll interval.

Messages arrive on the broker as soon as the drones report telemetry.

## Good to know

- Only assets of the connector's organization are sent, whatever `assetSns` lists. A serial number
  of another organization's asset sends nothing.
- The minimum interval limits how often a message is sent; it does not create messages. An asset that
  reports nothing (switched off, out of range) sends nothing.
- On Kafka, set the sink's message key to `{assetSn}`: the messages of one asset then stay on one
  partition, in order.
- On Kafka, create the topic before you start the bridge: the sink does not create topics. Until the
  topic exists, Integration Hub keeps the messages and retries them every 30 seconds, so none are lost.
- On OPC UA, the sink writes each payload field to the node whose alias has the same name, for example
  alias `batteryPercent` → `ns=2;s=BatteryPercent`. A nested field such as `position.latitude` needs a
  mapping that lifts it to the top level first.
- If the connection to the platform drops, the source reconnects by itself, waiting a little longer
  after each failed attempt (up to 30 seconds). Messages from the time it was disconnected are not
  sent afterwards.
- To change the assets, fields or interval, edit the connector and save a new version, then stop and
  start the bridge so it uses the new config.
