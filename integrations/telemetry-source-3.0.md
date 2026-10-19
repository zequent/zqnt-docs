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
- Create the connector signed in as a user of the organization whose assets you want to forward. A
  telemetry connector always belongs to an organization and only ever sends that organization's
  assets. A connector without an organization (for example one a system admin created) shows an
  error and sends nothing.

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
| `details` | Every device-specific value the asset reports, such as wind speed on a dock |
| `details.<key>` | One device-specific value, for example `details.wind_speed` |

`assetSn`, `assetId` and `observedAt` are always sent, whatever you pick. A value the asset did not
report is left out of the message rather than sent as zero.

## What a message looks like

Each message is one sample of one asset:

```json
{
  "assetSn": "DRONE-7",
  "assetId": "6f1c2a0e-3b7d-4c55-9f0e-2d8a1b4c7e90",
  "observedAt": "2026-10-19T09:41:07.250Z",
  "position": { "latitude": 52.5201, "longitude": 13.4049, "altitude": 81.3 },
  "relativeAltitude": 40.0,
  "headingDegrees": 271.5,
  "batteryPercent": 76
}
```

A bridge with a mapping that has no rules passes this through unchanged. A mapping with rules sends
only the fields it maps, so map `assetSn` too if the sink needs it, for example in an MQTT topic.

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
   username and password in a vault and select it in the connector form.

3. **Mapping.** Create a mapping named `pass-through` and add no rules.

4. **Bridge.** Under **Bridges**, create a bridge from `fleet-telemetry` to `control-room-mqtt` with
   the `pass-through` mapping and start it. A telemetry source pushes its messages, so the bridge
   has no poll interval.

Messages arrive on the broker as soon as the drones report telemetry.

## Good to know

- Only assets of the connector's organization are sent, whatever `assetSns` lists. A serial number
  of another organization's asset sends nothing.
- The minimum interval limits how often a message is sent; it does not create messages. An asset that
  reports nothing (switched off, out of range) sends nothing.
- If the connection to the platform drops, the source reconnects by itself, waiting a little longer
  after each failed attempt (up to 30 seconds). Messages from the time it was disconnected are not
  sent afterwards.
- To change the assets, fields or interval, edit the connector and save a new version, then stop and
  start the bridge so it uses the new config.
