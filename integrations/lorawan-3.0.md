# Connecting an on-prem LoRaWAN network server (3.0 preview)

Your LoRaWAN sensors — door and window contacts, motion detectors, fence and vibration sensors,
temperature and humidity sensors, level sensors, GPS trackers — can appear in Zequent as assets, and
their alarms can start Applications. You keep your own network server (for example ChirpStack) on your
premises; the Integration Hub subscribes to its MQTT integration.

```
sensors ──radio──▶ gateways ──▶ network server (ChirpStack) ──MQTT──▶ Integration Hub
                                                                        │ decoder per device profile
                                                                        ▼
                                              asset per DevEUI · telemetry · position · alerts
                                                                        │
                                                                        ▼
                                                  Integration event triggers start Applications
```

This page describes a preview; details can change until 3.0 is released.

## Before you start

- The Integration Hub is connected to the platform (`ZQNT_PLATFORM_ENABLED=true` and an integration
  credential in `ZQNT_EDGE_TOKEN`, see [Zequent Platform
  Integration](integration-hub.md#backend----zequent-platform-integration-optional)). It creates the
  sensors' assets in Connector and publishes their readings to the Live Data Service
  (`ZQNT_LIVE_DATA_HOST` / `ZQNT_LIVE_DATA_PORT`).
- The Integration Hub can reach the network server's MQTT broker. ChirpStack v4 ships with Mosquitto;
  its MQTT integration publishes every uplink on `application/<application id>/device/<DevEUI>/event/up`
  as JSON (the default *JSON* marshaler; *Protobuf* is not supported).
- An MQTT user for the Integration Hub on that broker that may subscribe to the uplink topics. Read
  access is all it needs.
- Everything you create — connectors, scripts, the bridge — belongs to one organization, and so do
  the sensors' assets. Create them signed in as a user of that organization; a system admin picks the
  organization in each form.

## 1. Store the broker credentials

Under **Manage → Access & Integrations → Secrets → Integration Hub**, create a **Username / Password**
secret with the MQTT user. If the broker asks for a client certificate, create a **Certificate**
secret as well.

## 2. The source: your network server

Under **Manage → Integrations → Connectors**, create a connector and pick the template **LoRaWAN
network server (ChirpStack v4)**. It is an **MQTT** connector with role **Source**:

| Setting | Config key | What it does |
| --- | --- | --- |
| Broker | `broker` | `tcp://host:1883`, or `ssl://host:8883` for TLS |
| Client id | `clientId` | How the Integration Hub identifies itself to the broker. Keep it fixed and unique |
| Topics | `topics` | Topic filters, `+` and `#` allowed. The template subscribes to every uplink: `application/+/device/+/event/up`. Name one application instead of `+` to take only its devices |
| QoS | `qos` | `1` (the template) makes the broker resend a message the Hub did not acknowledge |
| Persistent session | `persistentSession` | With QoS 1, the broker keeps the uplinks that arrive while the Hub restarts and delivers them afterwards |
| Secret | `vaultId` | The username/password secret from step 1 |
| Client certificate | `tlsVaultId` | The certificate secret, only if the broker asks for one |
| CA certificate | `caCertificate` | The PEM of your own certification authority, when the broker's certificate is not signed by a public one |

When the broker cannot be reached, the connector shows an error right away. A connection lost later
is re-established by itself, waiting a little longer after each failed attempt (up to two minutes),
and the topics are subscribed again.

## 3. A decoder per device profile

A LoRaWAN sensor sends a few bytes; what they mean depends on the device. A **decoder** turns them
into a reading: which device, what it measured, and which alarms it raises. Decoders are scripts:
under **Manage → Integrations → Scripts**, create a script and pick a template:

| Template | Layout | Telemetry | Alarms (code) |
| --- | --- | --- | --- |
| Decoded by the network server (ChirpStack v4) | ChirpStack's own device-profile codec (`object`) | every field of the object; `latitude`/`longitude` become the position | `lorawan.sensor.alarm` when the object has `alarm: true` |
| Door/contact sensor | Dragino LDS02 | `open`, `openCount`, `lastOpenMinutes`, `batteryVoltage` | `lorawan.door.open`, `lorawan.door.open_too_long` |
| Motion sensor | Elsys ERS / ERS Eye | `motion`, `occupancy`, `temperature`, `humidity`, … | `lorawan.motion.detected` |
| Fence/vibration sensor | a plain 4-byte layout to adapt | `vibration`, `tamper`, `cut`, `vibrationPeakMg` | `lorawan.fence.cut`, `lorawan.fence.tamper` (critical), `lorawan.fence.vibration` |
| Temperature/humidity sensor | Elsys ERS / ELT | `temperature`, `humidity`, … | `lorawan.temperature.high` / `.low`, `lorawan.humidity.high` once you set the limits at the top of the script |
| Level sensor | Dragino LDDS75 | `distanceMm`, `levelPercent`, `temperature` | `lorawan.level.low` / `.high`; set the mounting height and limits at the top of the script |
| GPS tracker | Dragino LGT-92 | position, `batteryVoltage`, `sos` | `lorawan.tracker.sos` (critical) |

Every reading also carries the radio values `rssi` and `snr` (of the best gateway), `fPort` and
`fCnt`, and uses the network server's receive time as the time it occurred.

If your sensor model has a different byte layout, start from the closest template and change its
`formatter` function; **Test** runs it on a sample uplink. If ChirpStack already decodes the payload
with a codec in the device profile, use the *Decoded by the network server* template.

## 4. The sink: devices become assets

Create a second connector from the template **LoRaWAN devices → assets**. It is a **Zequent Asset**
connector with role **Sink**:

| Setting | Config key | What it does |
| --- | --- | --- |
| Create an asset for a new device | `createAssets` | On (default): the first uplink of a device without an asset creates one in the bridge's organization. Off: uplinks of devices without an asset are dropped |
| Serial prefix | `serialPrefix` | A created asset's serial number is the prefix and the DevEUI, e.g. `LORAWAN-A84041000181C4F1` |
| Asset type | `assetType` | The type of a created asset (sensor by default). A custom decoder may set `assetType` in its reading to override it |
| Devices mapped to existing assets | `devices` | DevEUI → serial number of an asset that already exists, e.g. a gate you created by hand. Such a device is never created |

A created asset is named after the device's name in the network server, with the device profile as
its model and the DevEUI as its external id. An asset of another organization is never written to,
whatever the mapping says.

## 5. The bridge

Create a bridge with the source from step 2 and the sink from step 4. It needs no poll interval: the
broker pushes each uplink. Then give it one **pipeline per device profile**:

1. A **rule** step *matches* on `data.deviceInfo.deviceProfileName`, with the name of the device
   profile in ChirpStack (for example `LDS02`). Only that profile's uplinks pass this pipeline.
2. No mapping: the decoder reads the uplink as it arrives.
3. The decoder script from step 3 for that profile.

With one device profile only, a single pipeline with its decoder and no rule is enough.

From the first uplink on, the devices appear under **Assets**, with their telemetry, battery and
position (trackers), and their alarms in the alert list.

## 6. Alarms start Applications

Each reading goes to the bridge's **Integration** event triggers (**Manage → Event Triggers**, type
**Integration bridge**, see [Starting an Application from an
event](integration-hub-bridges-3.0.md#starting-an-application-from-an-event-triggers)). An alarm is
handed over with its code in the field `alarm`, one message per alarm:

- **Field** `alarm`, **equals** `lorawan.fence.cut` — start the intrusion response when the fence is
  cut.
- **Field** `temperature`, **greater than** `40` — the telemetry values are fields too, so a trigger
  can watch a reading without an alarm.

The message also carries `assetSn`, `deviceId` (the DevEUI), `deviceName`, `sensorType`,
`occurredAt` and, for a tracker, `latitude`/`longitude`; the Application receives them as inputs,
so a step can fly to the tracker or look at the gate (`$.input.assetSn`). A trigger limited to a site
fires only for readings with a position inside the site.

**Late uplinks.** A network server that was offline delivers its backlog when it comes back. Set a
**maximum event age** on the trigger (for example 5 minutes) so an alarm from an hour ago is shown on
the asset but starts nothing; it is measured from the time the uplink was received by the network
server, not from when it reached Zequent. A cooldown keeps a sensor that reports the same alarm with
every uplink from starting a run each time.

## Troubleshooting

| What you see | Why |
| --- | --- |
| The source connector shows an error | The broker address, port, user or certificate is wrong, or the broker is not reachable from the Integration Hub |
| No messages on the bridge | The topic does not match (check the application id), or the MQTT user may not subscribe to it |
| The bridge logs *not a LoRaWAN uplink* | The message is not a ChirpStack v4 JSON uplink, e.g. a `join` or `status` event on a broader topic filter |
| The bridge logs *the … layout needs N bytes* | The decoder does not fit the device: pick the right template for the profile, or a rule so other profiles do not reach it |
| No asset appears | *Create an asset for a new device* is off and the device is not mapped, or the bridge belongs to no organization |
| Alarms on the asset, but nothing starts | No Integration trigger on this bridge matches, the cooldown is running, or the alarm is older than the trigger's maximum event age (the operational events say why it did not start) |
