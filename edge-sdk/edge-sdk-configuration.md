# Edge SDK -- Configuration Guide

## Overview

The Zequent Edge SDK is configured primarily through Quarkus `application.properties` and environment variables. Configuration covers the edge identity, gRPC service endpoints, MQTT broker settings (for adapters that use MQTT), and operational parameters.

## Table of Contents

- [Configuration Methods](#configuration-methods)
- [Edge Identity Configuration](#edge-identity-configuration)
- [gRPC Client Configuration](#grpc-client-configuration)
- [gRPC Server Configuration](#grpc-server-configuration)
- [MQTT Configuration](#mqtt-configuration)
- [Object Storage Configuration](#object-storage-configuration)
- [Monitoring and Observability](#monitoring-and-observability)
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
| `quarkus.grpc.clients.live-data-service.host` | `QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST` |

---

## Edge Identity Configuration

These properties identify your edge adapter instance to the platform. They are mapped through the `EdgeClientConfig` interface using Quarkus `@ConfigMapping(prefix = "zequent.edge")`.

| Property | Environment Variable | Required | Description |
|----------|---------------------|----------|-------------|
| `zequent.edge.endpoint` | `EDGE_ADAPTER_TARGET_ENDPOINTS` | Yes | The address this adapter is reachable at (host:port) |
| `zequent.edge.sn` | `ZEQUENT_EDGE_SN` | Yes | Serial number of the managed device |
| `zequent.edge.asset-type` | `ZEQUENT_EDGE_ASSET_TYPE` | Yes | Asset type enum value (see below) |
| `zequent.edge.asset-vendor` | `ZEQUENT_EDGE_ASSET_VENDOR` | Yes | Asset vendor enum value (see below) |

### Asset Type Values

| Value | Description |
|-------|-------------|
| `ASSET_TYPE_DOCK` | Docking station |
| `ASSET_TYPE_DRONE` | Standalone drone |
| `ASSET_TYPE_VEHICLE` | Ground vehicle |
| `ASSET_TYPE_RC` | Remote controller |
| `ASSET_TYPE_UNKNOWN` | Unknown type |

### Asset Vendor Values

| Value | Description |
|-------|-------------|
| `DJI` | DJI |
| `VENDOR_UNKNOWN` | Unknown vendor |

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
zequent.edge.asset-vendor=DJI
```

---

## gRPC Client Configuration

The edge adapter connects to platform services (Live Data, Connector) via gRPC. Each client is configured using the standard Quarkus gRPC client properties.

The adapter connects to three platform services. In local dev, set host and port directly. In Docker/Kubernetes, service discovery is handled via Stork (see below).

| Service | Environment Variable (Host) | Default Host | Environment Variable (Port) | Default Port |
|---------|----------------------------|--------------|----------------------------|--------------|
| Live Data | `LIVE_DATA_SERVICE_HOST` | `localhost` | `LIVE_DATA_SERVICE_PORT` | `8003` |
| Connector | `CONNECTOR_SERVICE_HOST` | `localhost` | `CONNECTOR_SERVICE_PORT` | `8010` |
| Mission Autonomy | `MISSION_AUTONOMY_SERVICE_HOST` | `localhost` | `MISSION_AUTONOMY_SERVICE_PORT` | `8004` |

### Local Development

```properties
quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003

quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010

quarkus.grpc.clients.mission-autonomy-service.host=localhost
quarkus.grpc.clients.mission-autonomy-service.port=8004
```

### Stork-based Service Discovery (Docker / Kubernetes)

In `%docker` and `%k8s` profiles, gRPC clients use [Stork](https://smallrye.io/smallrye-stork/) for service discovery instead of direct host/port configuration.

**Docker (static list):**

```properties
%docker.quarkus.grpc.clients.connector-service.name-resolver=stork
%docker.stork.connector-service.service-discovery.type=static
%docker.stork.connector-service.service-discovery.address-list=${CONNECTOR_SERVICE_HOST:connector-service}:${CONNECTOR_SERVICE_PORT:8010}
%docker.stork.connector-service.load-balancer.type=round-robin
```

**Kubernetes (dynamic discovery):**

```properties
%k8s.quarkus.grpc.clients.connector-service.name-resolver=stork
%k8s.stork.connector-service.service-discovery.type=kubernetes
%k8s.stork.connector-service.service-discovery.k8s-namespace=default
%k8s.stork.connector-service.service-discovery.application=connector-service
%k8s.stork.connector-service.service-discovery.refresh-period=5s
%k8s.stork.connector-service.load-balancer.type=round-robin
```

The same pattern applies for `live-data-service` and `mission-autonomy-service`.

---

## MQTT Configuration

Some adapters use MQTT to communicate with the physical device — the DJI adapter uses it for OSD telemetry, service commands, and state updates. This is not a requirement of the Edge SDK itself: other built-in adapters talk to their device over whatever transport the vendor actually uses (serial for Betaflight, MAVLink over serial/UDP for MAVSDK-based adapters, raw TCP for Sapient) and have no MQTT configuration at all. If your custom adapter does use MQTT, it's configured through the SmallRye Reactive Messaging MQTT connector as below.

### Broker Configuration

| Property | Environment Variable | Description |
|----------|---------------------|-------------|
| `zequent.mqtt.broker.host` | `MQTT_BROKER_HOST` | MQTT broker hostname |
| `zequent.mqtt.broker.username` | `MQTT_DOCK_USERNAME` | MQTT username for dock communication |
| `zequent.mqtt.broker.password` | `MQTT_DOCK_PASSWORD` | MQTT password for dock communication |

The reactive messaging channels use separate credentials for the cloud backend connection:

| Environment Variable | Description |
|---------------------|-------------|
| `MQTT_USERNAME` | Username for cloud messaging channels |
| `MQTT_PASSWORD` | Password for cloud messaging channels |
| `MQTT_BROKER_PORT` | MQTT broker port (default: `8883`) |

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
    image: ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0
    env_file:
      - .env
    ports:
      - "9001:9001"
```

Set `EDGE_ADAPTER_TARGET_ENDPOINTS`, service host/port values, device identity, and device-specific broker credentials in `.env`.

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
        image: ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0
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
        - name: QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST
          value: "live-data-service"
        - name: QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_HOST
          value: "connector-service"
```

---

## Troubleshooting

### Problem: Adapter cannot connect to platform services

**Check 1:** Verify gRPC client settings:

```bash
echo $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST
echo $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT
```

**Check 2:** Test network connectivity:

```bash
nc -zv $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT
```

**Check 3:** Look for connection errors in the logs:

```
gRPC stream failed for device XXXXX: UNAVAILABLE
```

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
echo $ZEQUENT_MQTT_BROKER_HOST
```

**Check 2:** Check that MQTT topics match the device model's expected patterns.

**Check 3:** Look for MQTT connection logs on startup.

### Problem: Configuration not taking effect

**Check 1:** Environment variables take precedence over `application.properties`. Verify no conflicting env vars are set.

**Check 2:** Quarkus caches configuration at startup. Restart the application after changing properties.

**Check 3:** Restart the adapter container after changing container environment variables.
