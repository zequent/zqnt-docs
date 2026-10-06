# Edge SDK — Configuration Guide

## Overview

The Zequent Edge SDK is configured primarily through Quarkus `application.properties` and environment variables. Configuration covers the edge identity, gRPC service endpoints, MQTT broker settings (for adapters that use MQTT), and operational parameters.

## Table of Contents

- [Configuration Methods](#configuration-methods)
- [Edge Identity Configuration](#edge-identity-configuration)
- [gRPC Client Configuration](#grpc-client-configuration)
- [MQTT Configuration](#mqtt-configuration)
- [Object Storage Configuration](#object-storage-configuration)
- [Environment-Specific Examples](#environment-specific-examples)
- [Troubleshooting](#troubleshooting)

---

## Configuration Methods

### Priority Order (highest to lowest)

1. **Environment Variables** (e.g., `ZEQUENT_EDGE_SN`)
2. **System Properties** (e.g., `-Dzequent.edge.sn=...`)
3. **.env file** (automatically loaded by Quarkus)
4. **application.properties** (defaults)

Quarkus maps property names to environment variable names by converting to uppercase and replacing dots and hyphens with underscores. For example:

| Property | Environment Variable |
|----------|---------------------|
| `zequent.edge.sn` | `ZEQUENT_EDGE_SN` |
| `zequent.edge.asset-type` | `ZEQUENT_EDGE_ASSET_TYPE` |
| `grpc.client.live-data.host` | `GRPC_CLIENT_LIVE_DATA_HOST` |

---

## Edge Identity Configuration

These properties identify your edge adapter instance to the platform, populated from `application.properties` (with `${VAR:default}` substitution) into `EdgeClientConfig` — a plain builder class (`@Data @Builder`), not a Quarkus `@ConfigMapping` interface; there is no `@ConfigMapping` usage anywhere in the SDK.

| Property | Environment Variable | Required | Description |
|----------|---------------------|----------|-------------|
| `zequent.edge.endpoint` | `EDGE_ADAPTER_TARGET_ENDPOINTS` | Yes | The address this adapter is reachable at (host:port) |
| `zequent.edge.sn` | `ZEQUENT_EDGE_SN` | Yes | Serial number of the managed device |
| `zequent.edge.asset-type` | `ZEQUENT_EDGE_ASSET_TYPE` | Yes | Asset type enum value (see below) |
| `zequent.edge.asset-vendor` | `ZEQUENT_EDGE_ASSET_VENDOR` | Yes | Asset vendor enum value (see below) |

### Asset Type Values

Confirmed against the real proto contract (`AssetTypeEnum`, mirrored 1-to-1 by every SDK):

| Value | Description |
|-------|-------------|
| `ASSET_TYPE_UNKNOWN` | Unknown type |
| `ASSET_TYPE_AIRCRAFT` | Drone/aircraft |
| `ASSET_TYPE_DOCK` | Docking station |
| `ASSET_TYPE_SENSOR` | Sensor node |
| `ASSET_TYPE_CAMERA` | Standalone camera |
| `ASSET_TYPE_OTHER` | Anything else |
| `ASSET_TYPE_JAMMER` | RF jammer |
| `ASSET_TYPE_CYBER_ATTACK` | Cyber-attack asset |
| `ASSET_TYPE_SAPIENT` | SAPIENT-protocol node |
| `ASSET_TYPE_RNS` | Reticulum (RNS) node |

`ASSET_TYPE_DRONE`/`ASSET_TYPE_VEHICLE`/`ASSET_TYPE_RC` do not exist — use `ASSET_TYPE_AIRCRAFT` for
a drone.

### Asset Vendor Values

| Value | Description |
|-------|-------------|
| `ASSET_VENDOR_DJI` | DJI |
| `ASSET_VENDOR_AUTEL` | Autel |
| `ASSET_VENDOR_ROS` | ROS-based |
| `ASSET_VENDOR_MAVLINK` | MAVLink/MAVSDK (PX4, ArduPilot) |
| `ASSET_VENDOR_RTMP_RTSP` | Generic RTMP/RTSP video source |
| `ASSET_VENDOR_SAPIENT` | SAPIENT-protocol node |
| `ASSET_VENDOR_BETAFLIGHT` | Betaflight flight controller |
| `ASSET_VENDOR_RNS` | Reticulum (RNS) node |

There is no `VENDOR_UNKNOWN`/`ASSET_VENDOR_UNKNOWN` value — every vendor must be one of the above.
Every value requires the full `ASSET_TYPE_`/`ASSET_VENDOR_` prefix, not the bare enum member name —
a bare value fails lookup and is rejected.

### Profile-specific Endpoint

The `zequent.edge.endpoint` property is typically set per profile to reflect the correct address for each environment:

```properties
# Dev: direct local address
%dev.zequent.edge.endpoint=localhost:9001

# Docker: configurable via env, defaults to service name
%docker.zequent.edge.endpoint=${EDGE_ADAPTER_TARGET_ENDPOINTS:edge-adapter-dji:9001}

# Kubernetes: use K8s service name
%k8s.zequent.edge.endpoint=edge-adapter-dji
```

### Example

```properties
zequent.edge.sn=YOUR_DEVICE_SN
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=ASSET_VENDOR_DJI
```

---

## gRPC Client Configuration

Every call to the platform carries the adapter's edge credential, `ZQNT_EDGE_TOKEN` — issued in the Admin
Console under **Manage → Access & Integrations → Credentials** (kind *Edge adapter*, or *Integration Hub* for the
hub), or offline with `core/scripts/mint-edge-credential.py`. It reaches only the device-facing calls. The adapter
verifies the platform's calls into it with `ZQNT_PLATFORM_PUBLIC_KEY` (alias `SERVICE_AUTH_PUBLIC_KEY`); without
it every platform command is refused. `ZQNT_EDGE_AUTH_DISABLED=true` turns that check off, for a local simulator
stack only. A customer application uses a *client* credential instead (`ZQNT_CLIENT_TOKEN`, see the client SDK
configuration).

The SDK does not open connections to the platform itself: your adapter creates a gRPC channel per service,
with the edge credential on it, and hands it to the SDK service (see
[Quickstart — Wire the SDK](edge-sdk-quickstart.md#step-3b-wire-the-sdk)). The DJI adapter and the quickstart read
each address from `grpc.client.<service>.host` / `.port`, which default to these environment variables:

| Service | Environment Variable (Host) | Default Host | Environment Variable (Port) | Default Port |
|---------|----------------------------|--------------|----------------------------|--------------|
| Live Data | `LIVE_DATA_SERVICE_HOST` | `localhost` | `LIVE_DATA_SERVICE_PORT` | `8003` |
| Connector | `CONNECTOR_SERVICE_HOST` | `localhost` | `CONNECTOR_SERVICE_PORT` | `8010` |
| Mission Autonomy | `MISSION_AUTONOMY_SERVICE_HOST` | `localhost` | `MISSION_AUTONOMY_SERVICE_PORT` | `8004` |

```properties
grpc.client.live-data.host=${LIVE_DATA_SERVICE_HOST:localhost}
grpc.client.live-data.port=${LIVE_DATA_SERVICE_PORT:8003}

grpc.client.connector.host=${CONNECTOR_SERVICE_HOST:localhost}
grpc.client.connector.port=${CONNECTOR_SERVICE_PORT:8010}

grpc.client.mission-autonomy.host=${MISSION_AUTONOMY_SERVICE_HOST:localhost}
grpc.client.mission-autonomy.port=${MISSION_AUTONOMY_SERVICE_PORT:8004}
```

In Docker or Kubernetes, set the host variables to the services' DNS names (e.g. `connector-service`). There is
no Stork service discovery.

---

## MQTT Configuration

Some adapters use MQTT to communicate with the physical device — the DJI adapter uses it for OSD telemetry, service commands, and state updates. This is not a requirement of the Edge SDK itself: other built-in adapters talk to their device over whatever transport the vendor actually uses (serial for Betaflight, MAVLink over serial/UDP for MAVSDK-based adapters, raw TCP for Sapient) and have no MQTT configuration at all. If your custom adapter does use MQTT, it's configured through the SmallRye Reactive Messaging MQTT connector as below.

### Broker Configuration

Confirmed against the real DJI adapter's own `application.properties` — every one of these carries a
`ZQNT_` prefix, not a bare `MQTT_`/`ZEQUENT_MQTT_` one:

| Property | Environment Variable | Description |
|----------|---------------------|-------------|
| `zequent.mqtt.broker.host` | `ZQNT_MQTT_BROKER_HOST` | MQTT broker hostname |
| `zequent.mqtt.broker.username` | `ZQNT_MQTT_DOCK_USERNAME` | MQTT username for direct dock communication |
| `zequent.mqtt.broker.password` | `ZQNT_MQTT_DOCK_PASSWORD` | MQTT password for direct dock communication |

The reactive messaging channels use separate credentials for the cloud backend connection:

| Environment Variable | Description |
|---------------------|-------------|
| `ZQNT_MQTT_USERNAME` | Username for cloud messaging channels |
| `ZQNT_MQTT_PASSWORD` | Password for cloud messaging channels |
| `ZQNT_MQTT_BROKER_PORT` | MQTT broker port (default: `8883`) |

### Channel Configuration Pattern

Each MQTT channel (incoming or outgoing) follows this pattern:

```properties
# Incoming channel
mp.messaging.incoming.<channel-name>.connector=smallrye-mqtt
mp.messaging.incoming.<channel-name>.topic=<mqtt/topic/pattern>
mp.messaging.incoming.<channel-name>.host=<broker-host>
mp.messaging.incoming.<channel-name>.port=8883
mp.messaging.incoming.<channel-name>.username=<username>
mp.messaging.incoming.<channel-name>.password=<password>
mp.messaging.incoming.<channel-name>.ssl=true

# Outgoing channel
mp.messaging.outgoing.<channel-name>.connector=smallrye-mqtt
mp.messaging.outgoing.<channel-name>.host=<broker-host>
mp.messaging.outgoing.<channel-name>.port=8883
mp.messaging.outgoing.<channel-name>.username=<username>
mp.messaging.outgoing.<channel-name>.password=<password>
mp.messaging.outgoing.<channel-name>.ssl=true
```

### Typical Channels for a DJI Adapter

| Channel Name | Direction | Topic Pattern | Purpose |
|-------------|-----------|---------------|---------|
| `osd` | Incoming | `thing/product/+/osd` | On-screen display / telemetry |
| `state` | Incoming | `thing/product/+/state` | Device state changes |
| `status` | Incoming | `sys/product/+/status` | Topology updates |
| `status_reply` | Outgoing | (dynamic) | Topology update replies |
| `cloud-to-dock` | Outgoing | (dynamic) | Commands sent to the dock |
| `services-reply` | Incoming | `thing/product/+/services_reply` | Replies to service commands |
| `requests` | Incoming | `thing/product/+/requests` | Device-initiated requests |
| `drc-up` | Incoming | `thing/product/+/drc/up` | DRC (Direct Remote Control) upstream data |

---

## Object Storage Configuration

If your adapter needs to upload files (e.g., KMZ flight plans) to object storage:

| Property | Environment Variable | Description |
|----------|---------------------|-------------|
| `storage.username` | `S3_USERNAME` | S3 user identifier |
| `storage.endpoint` | `S3_ENDPOINT` | S3-compatible endpoint URL |
| `storage.access-key` | `S3_ACCESS_KEY` | Access key |
| `storage.secret-key` | `S3_SECRET_KEY` | Secret key |
| `storage.region` | `S3_REGION` | Storage region |
| `storage.bucket` | `S3_BUCKET` | Target bucket name |
| `storage.object-key-prefix` | `S3_OBJECT_KEY_PREFIX` | Prefix for all object keys |

---

## Environment-Specific Examples

### Local Development

```properties
zequent.edge.sn=YOUR_DEVICE_SN
zequent.mqtt.broker.host=your-broker.example.com
```

The gRPC client endpoints default to `localhost` on their respective ports in dev mode, so no extra configuration is needed unless the services run on different hosts.

### Docker Compose

Use the deployment-local `.env` file for adapter configuration:

```yaml
services:
  edge-adapter:
    image: ghcr.io/zequent/zqnt-edge-adapter-dji:2.0.0
    env_file:
      - .env
    ports:
      - "9001:9001"
```

Set `EDGE_ADAPTER_TARGET_ENDPOINTS`, `ZQNT_EDGE_TOKEN`, `ZQNT_PLATFORM_PUBLIC_KEY`, service host/port values, device identity, and device-specific broker credentials in `.env`.

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-dji
spec:
  template:
    spec:
      containers:
      - name: edge-dji
        image: ghcr.io/zequent/zqnt-edge-adapter-dji:2.0.0
        ports:
        - containerPort: 9001
        env:
        - name: ZEQUENT_EDGE_ENDPOINT
          value: "edge-dji:9001"
        - name: ZEQUENT_EDGE_SN
          valueFrom:
            secretKeyRef:
              name: edge-secrets
              key: device-sn
        - name: ZQNT_EDGE_TOKEN
          valueFrom:
            secretKeyRef:
              name: edge-secrets
              key: edge-token
        - name: ZQNT_PLATFORM_PUBLIC_KEY
          valueFrom:
            secretKeyRef:
              name: edge-secrets
              key: platform-public-key
        - name: LIVE_DATA_SERVICE_HOST
          value: "live-data-service"
        - name: CONNECTOR_SERVICE_HOST
          value: "connector-service"
```

---

## Troubleshooting

### Problem: Adapter cannot connect to platform services

**Check 1:** Verify gRPC client settings:

```bash
echo $LIVE_DATA_SERVICE_HOST
echo $LIVE_DATA_SERVICE_PORT
```

**Check 2:** Test network connectivity:

```bash
nc -zv $LIVE_DATA_SERVICE_HOST $LIVE_DATA_SERVICE_PORT
```

**Check 3:** Look for connection errors in the logs:

```
gRPC stream failed for device XXXXX: UNAVAILABLE
```

### Problem: Calls fail with `UNAUTHENTICATED`

**Calls from the adapter to the platform:** `ZQNT_EDGE_TOKEN` is missing, revoked, or not attached to the
channel. Issue a new edge credential in the Admin Console.

**Calls from the platform into the adapter:** `ZQNT_PLATFORM_PUBLIC_KEY` is missing or is not the key of the
platform that is calling. The adapter logs `ZQNT_PLATFORM_PUBLIC_KEY is not set` at startup when it is missing.

### Problem: Telemetry not arriving at the platform

**Check 1:** Verify the device serial number matches what is configured:

```bash
echo $ZEQUENT_EDGE_SN
```

**Check 2:** Look for telemetry stream logs:

```
Started gRPC telemetry stream for device XXXXX
Telemetry response received for device XXXXX
```

**Check 3:** Ensure the Live Data Service is running and reachable.

### Problem: MQTT messages not being received

**Check 1:** Verify MQTT broker connection:

```bash
echo $ZQNT_MQTT_BROKER_HOST
```

**Check 2:** Check that MQTT topics match the device model's expected patterns.

**Check 3:** Look for MQTT connection logs on startup.

### Problem: Configuration not taking effect

**Check 1:** Environment variables take precedence over `application.properties`. Verify no conflicting env vars are set.

**Check 2:** Quarkus caches configuration at startup. Restart the application after changing properties.

**Check 3:** Restart the adapter container after changing container environment variables.
