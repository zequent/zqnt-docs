# Connecting a TAK Server (3.0 preview)

Zequent and TAK users (ATAK, WinTAK, iTAK) can share one picture. TAK users and the units they
share appear as assets on the operations map, the hostile and unknown objects they mark appear as
detections, and their emergency alerts and GeoChat messages reach the operators. In the other
direction, TAK users see your drones, docks and sensors, what your sensors detect, and your alerts
as GeoChat messages.

```
ATAK / WinTAK / iTAK ──▶ TAK Server ──CoT over TLS──▶ Integration Hub ──▶ assets · detections · alarms
                                    ◀──CoT over TLS── Integration Hub ◀── positions · detections · alerts
```

The Integration Hub talks Cursor-on-Target (CoT) XML to the TAK Server's streaming port, as one more
TAK client with a client certificate. This page describes a preview; details can change until 3.0
is released.

## Before you start

- The Integration Hub is connected to the platform (`ZQNT_PLATFORM_ENABLED=true`, see [Zequent Platform
  Integration](integration-hub.md#backend----zequent-platform-integration-optional)).
- The Integration Hub can reach the TAK Server's streaming port, usually `8089` (TLS). A TAK Server
  in another network needs a route or a VPN to it.
- A **client certificate** for the Integration Hub, issued by the TAK Server's certificate authority
  (on the TAK Server, for example `makeCert.sh client zequent`), and the CA certificate. TAK Servers
  hand certificates out as `.p12` files; convert them to PEM:

  ```bash
  openssl pkcs12 -in zequent.p12 -clcerts -nokeys -out zequent.crt.pem
  openssl pkcs12 -in zequent.p12 -nocerts -nodes -out zequent.key.pem
  openssl pkcs12 -in truststore-root.p12 -cacerts -nokeys -out tak-ca.pem
  ```

- Give the certificate's user the TAK Server groups whose users and data Zequent may see; TAK users
  see Zequent's data in those groups.
- Everything you create — connectors, the bridges — belongs to one organization, and so do the TAK
  users' assets. Create them signed in as a user of that organization; a system admin picks the
  organization in each form.

## 1. Store the certificate

Under **Manage → Access & Integrations → Secrets → Integration Hub**, create a **Certificate**
secret with the client certificate, its private key and the TAK Server's CA certificate.

## 2. TAK users in Zequent

Under **Manage → Integrations → Connectors**, create a connector from the template **TAK Server →
Zequent** (a **TAK Server** connector with role **Source**), set the address and pick the
certificate:

| Setting | Config key | What it does |
| --- | --- | --- |
| TAK Server address | `address` | `tls://tak.example.com:8089`. `tcp://host:8087` connects without TLS (test servers only); `udp://239.2.3.1:6969` joins the mesh SA multicast group of TAK clients on the same network |
| Client certificate | `tlsVaultId` | The certificate secret from step 1 |
| CA certificate | `caCertificate` | Only when the secret carries no CA certificate |
| Types | `types` | Which CoT types are read. Default: every unit and marker (`a-`), GeoChat (`b-t-f`) and emergencies (`b-a-`) |
| Feed id | `feedId` | The device id of the TAK connection itself, which reports the marked objects. Default: the server's host name |
| Minimum interval | `minIntervalMs` | At most one position per TAK user per interval (default 1000 ms). Emergencies and chat always pass |

Then create a second connector from the template **TAK users → assets** (a **Zequent Asset** sink
with the serial prefix `TAK-`), and a bridge from the first to the second. The bridge needs no
mapping and no poll interval.

What arrives:

| From TAK | In Zequent |
| --- | --- |
| A TAK user's position, a friendly or assumed-friend unit (`a-f-…`, `a-a-…`) | An asset `TAK-<uid>` named after the callsign, with position, course, speed and battery; team, role and CoT type in its telemetry. Assets created this way have the type *other* and the model `TAK`; Zequent never sends commands to them |
| A hostile, suspect, unknown, neutral or pending object a TAK user marks (`a-h-…`, `a-s-…`, `a-u-…`, `a-n-…`, `a-p-…`) | A detection of the TAK feed's asset (`TAK-<feed id>`), with the class from the symbol (UAV, aircraft, person, vehicle, vessel) and the identity from its affiliation. Detection triggers and the map treat it like a sensor's detection; the platform records the identity as what this sensor says |
| An emergency: 911, ring the bell, troops in contact, geofence breached | A critical alert of the TAK user's asset (`tak.emergency.911`, `tak.emergency.ring_the_bell`, `tak.emergency.troops_in_contact`, `tak.emergency.geofence_breached`); a cancel is an info alert `tak.emergency.cancelled` |
| A GeoChat message | An info alert `tak.chat` of the sender's asset, with the text as its message |

Events that are already stale when they arrive are dropped, and so are Zequent's own events (uid
prefix `ZQNT-`).

Alerts go to the bridge's **Integration** event triggers like any device alarm (see [Connecting an
on-prem LoRaWAN network server](lorawan-3.0.md#6-alarms-start-applications)): for example **Field**
`alarm` **equals** `tak.emergency.911` starts a response flying a drone to the person
(`$.input.latitude`, `$.input.longitude`).

## 3. Zequent in TAK

Create a connector from the template **Zequent picture for TAK** (a **Zequent Telemetry** source
with *Detections* and *Alerts* switched on) and one from **Zequent → TAK Server** (a **TAK Server**
connector with role **Sink**, same address and certificate as in step 2), and a bridge from the
first to the second, without mapping or poll interval.

| Setting | Config key | What it does |
| --- | --- | --- |
| Callsign | `callsign` | The sender of Zequent's GeoChat messages (default `Zequent`) |
| Detections | `share.detections` | Detected objects as markers (default on). Objects taken for one of your own assets are left out: they already show as that asset |
| Identities | `share.identities` | Which detections are shared, by identity. Default: pending, unknown, neutral, suspect, hostile |
| Alerts as GeoChat | `share.alerts`, `share.minAlertSeverity` | Alerts of your assets as messages in the chat room, from warning upwards by default |
| Chat room | `chatRoom` | Default `All Chat Rooms` |
| Asset types, assets | `share.assetTypes`, `share.assetSns` | Share only these (default: all of the organization's assets) |
| Simulated assets | `share.simulated` | Off by default, so the simulator's drones never show on a real operation's map |
| Stale after | `staleSeconds` | How long TAK keeps showing a position after it was sent (default 60 s) |
| Minimum interval | `minIntervalMs` | At most one update per asset or object per interval (default 1000 ms) |

What TAK users see:

| From Zequent | In TAK |
| --- | --- |
| A drone | A friendly UAV (`a-f-A-M-F-Q`) named after the asset, with course, speed and battery |
| A dock | A friendly installation (`a-f-G-I`) |
| A sensor, camera, radar | Friendly sensor equipment (`a-f-G-E-S`) |
| Any other asset | A friendly ground symbol (`a-f-G`) |
| A detection | A marker with its identity and class, e.g. a hostile UAV `a-h-A-M-F-Q`, an unknown person `a-u-G-U-C-I`, with the detecting sensor and confidence in its remarks |
| An alert | A GeoChat message, e.g. `CRITICAL Gate north: Fence cut (lorawan.fence.cut)` |

Assets that came from TAK are never sent back, and nothing of another organization ever leaves the
platform.

## Mesh SA in the field

With the Integration Hub on the same network as the TAK clients (a field laptop on the operation's
Wi-Fi or MANET), use `udp://239.2.3.1:6969` as the address in both connectors, without a
certificate. Mesh SA is not encrypted: use it only on an operation network that is encrypted
underneath. Set the TAK clients' mesh to *TAK protocol version 0 (XML)*; the protobuf format of
version 1 is not read yet.

## Troubleshooting

| What you see | Why |
| --- | --- |
| The connector shows an error | The address or port is wrong, the TAK Server is not reachable, or it refused the certificate (wrong CA, expired, not a client certificate) |
| Connected, but no TAK users arrive | The certificate's user is in none of the groups the TAK users share with, or the bridge has no Zequent Asset sink |
| TAK users arrive, marked objects do not | Their CoT type is not in `types`, or they were stale when they arrived |
| Your drones do not show in TAK | The telemetry source streams other assets, the asset type is not shared, the asset is simulated, or the positions are older than *stale after* |
| Positions in TAK are a few dozen metres too high or low | TAK expects the height above the ellipsoid; Zequent sends the altitude above sea level |
