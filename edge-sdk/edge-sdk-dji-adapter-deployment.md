# DJI Edge Adapter -- Deployment Guide

The DJI Edge Adapter connects DJI docking stations and their sub-assets (drones) to the Zequent platform. It communicates with the dock via MQTT and exposes a gRPC interface toward the platform services.

---

## Prerequisites

- Access to the Zequent container registry (`ghcr.io/zequent`)
- A running MQTT broker reachable by the adapter (e.g. HiveMQ Cloud)
- Running Zequent platform services: Connector Service, Live Data Service, Mission Autonomy Service, Remote Control Service

---

## Supported DJI Firmware Versions

The DJI Edge Adapter has been tested and verified against the following DJI firmware major versions:

| Firmware Version | Status |
|-----------------|--------|
| v13.x.x | Supported |
| v14.x.x | Supported |
| v17.x.x | Supported |

> **Note:** Other firmware versions may work but are not officially validated. Ensure your DJI docking station and drone firmware are updated to one of the supported versions before deploying the adapter.

---

## Supported Device Models & Dynamic Capabilities

Beyond the fixed command set (takeoff, dock ops, etc.), the DJI adapter reports a per-device-model list of vendor-specific settings as dynamic [capabilities](../client-sdk/REMOTE_CONTROL.md#capabilities--custom-commands) — settable through `client.remoteControl().sendCustomCommand(...)` without the platform needing a dedicated method for each one. Which properties are exposed is baked into the adapter build (`dji-capabilities.yaml`), not runtime-configurable — adding a new model or property requires a custom adapter build.

As of this adapter version, two device profiles are defined:

| Model | Exposed properties |
|-------|---------------------|
| DJI Dock 3 | Fast photo transfer, dock silent mode, DJI user-experience program (disabled by default) |
| Matrice 4D / 4TD | Obstacle avoidance, height limit, night lights, distance limit, RTH altitude, FlyTo flight height/mode/link-loss action, return reserve battery, camera watermark, and (M4TD only) thermal camera palette/gain-mode/isotherm settings |

A dock or drone not matching either profile still works for standard flight/dock operations — it just reports no dynamic capabilities beyond those.

---

## Environment Variables

### Required

| Variable | Description |
|----------|-------------|
| `ZQNT_MQTT_BROKER_HOST` | MQTT broker hostname |
| `ZQNT_MQTT_USERNAME` | MQTT username for cloud backend channels |
| `ZQNT_MQTT_PASSWORD` | MQTT password for cloud backend channels |
| `ZQNT_MQTT_DOCK_USERNAME` | MQTT username for direct dock communication |
| `ZQNT_MQTT_DOCK_PASSWORD` | MQTT password for direct dock communication |
| `CONNECTOR_SERVICE_HOST` | Hostname of the Connector Service |
| `LIVE_DATA_SERVICE_HOST` | Hostname of the Live Data Service |
| `MISSION_AUTONOMY_SERVICE_HOST` | Hostname of the Mission Autonomy Service |
| `REMOTE_CONTROL_SERVICE_HOST` | Hostname of the Remote Control Service |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `ZQNT_MQTT_BROKER_PORT` | `8883` | MQTT broker port (TLS) |
| `CONNECTOR_SERVICE_PORT` | `8010` | Connector Service gRPC port |
| `LIVE_DATA_SERVICE_PORT` | `8003` | Live Data Service gRPC port |
| `MISSION_AUTONOMY_SERVICE_PORT` | `8004` | Mission Autonomy Service gRPC port |
| `REMOTE_CONTROL_SERVICE_PORT` | `8002` | Remote Control Service gRPC port |
| `EDGE_ADAPTER_TARGET_ENDPOINTS` | `edge-adapter-dji:9001` | Address at which this adapter is reachable by the platform |
| `REDIS_URL` | `redis://localhost:6379` | Redis connection used by the adapter's caching layer |
| `ZQNT_DOCK_OFFLINE_TIMEOUT` | `10s` | A dock is reported offline if no OSD telemetry arrives within this window |
| `ZQNT_DOCK_WATCHDOG_INTERVAL` | `2s` | How often the adapter checks each dock's telemetry age against `ZQNT_DOCK_OFFLINE_TIMEOUT` |
| `S3_ENDPOINT` | `https://s3.amazonaws.com` | S3-compatible storage endpoint |
| `S3_REGION` | `eu-central-1` | S3 region |
| `S3_BUCKET` | `zqnt` | S3 bucket name |
| `S3_OBJECT_KEY_PREFIX` | `zqnt` | Prefix for stored objects |
| `S3_USERNAME` | -- | S3 user identifier |
| `S3_ACCESS_KEY` | -- | S3 access key |
| `S3_SECRET_KEY` | -- | S3 secret key |

## Docker Compose

Use the same deployment-local `.env` file as the platform stack.

```yaml
services:
  edge-dji:
    image: ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0
    container_name: edge-adapter-dji
    env_file:
      - .env
    ports:
      - "9001:9001"
```

The **container** (not necessarily the service key) must be named `edge-adapter-dji` — the Admin
Console's adapter manager (`AdapterManagementService`) resolves the container to start/stop/restart
by building that exact name (`edge-adapter-` + vendor) and shells out to it directly; it does not
look this up any other way. Set `container_name` explicitly as shown — Docker Compose's default
container naming from a `edge-dji` service key would not produce this name on its own. The public
`docker-compose.customer.yml` template in this repo does not currently set `container_name` for its
`edge-dji` service, so the Admin Console's start/stop/restart controls will not find that container
as shipped.

Set the required MQTT, service endpoint, and optional storage values in `.env`.

---

## Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-adapter-dji
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edge-adapter-dji
  template:
    metadata:
      labels:
        app: edge-adapter-dji
    spec:
      containers:
        - name: edge-adapter-dji
          image: ghcr.io/zequent/zqnt-edge-adapter-dji:1.3.0
          ports:
            - containerPort: 9001
          env:
            - name: QUARKUS_PROFILE
              value: "k8s"
            - name: EDGE_ADAPTER_TARGET_ENDPOINTS
              value: "edge-dji:9001"
            - name: ZQNT_MQTT_BROKER_HOST
              value: "your-broker.example.com"
            - name: ZQNT_MQTT_BROKER_PORT
              value: "8883"
            - name: ZQNT_MQTT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-username
            - name: ZQNT_MQTT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-password
            - name: ZQNT_MQTT_DOCK_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-username
            - name: ZQNT_MQTT_DOCK_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-password
            - name: CONNECTOR_SERVICE_HOST
              value: "connector-service"
            - name: CONNECTOR_SERVICE_PORT
              value: "8010"
            - name: LIVE_DATA_SERVICE_HOST
              value: "live-data-service"
            - name: LIVE_DATA_SERVICE_PORT
              value: "8003"
            - name: MISSION_AUTONOMY_SERVICE_HOST
              value: "mission-autonomy-service"
            - name: MISSION_AUTONOMY_SERVICE_PORT
              value: "8004"
            - name: REMOTE_CONTROL_SERVICE_HOST
              value: "remote-control-service"
            - name: REMOTE_CONTROL_SERVICE_PORT
              value: "8002"
            # S3 (optional - required for mission file uploads)
            - name: S3_ENDPOINT
              value: "https://s3.amazonaws.com"
            - name: S3_REGION
              value: "eu-central-1"
            - name: S3_BUCKET
              value: "zqnt"
            - name: S3_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-username
            - name: S3_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-access-key
            - name: S3_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-secret-key
---
apiVersion: v1
kind: Service
metadata:
  name: edge-adapter-dji
spec:
  selector:
    app: edge-adapter-dji
  ports:
    - port: 9001
      targetPort: 9001
```

The `container_name` requirement above is specific to the Docker Compose path — the Admin Console's
adapter manager shells out to a configured container runtime (`docker`/`podman`) by that exact name,
which has no equivalent concept in Kubernetes. Start/stop/restart from the Admin Console is not
expected to work against a Kubernetes deployment of this adapter.

> **Note:** In Kubernetes deployments, keep credentials in Kubernetes Secrets and expose only the service hostnames required by the adapter.

---

## MQTT Topics

The adapter subscribes and publishes to the following MQTT topics. The `+` wildcard matches the device serial number.

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `thing/product/+/osd` | Incoming | Drone telemetry (OSD data) |
| `thing/product/+/state` | Incoming | Device state updates |
| `thing/product/+/events` | Incoming | Device-reported events (e.g. flight-task lifecycle) — feeds task progress/asset status notifications back to the platform |
| `sys/product/+/status` | Incoming | Dock/drone topology updates |
| `thing/product/+/services_reply` | Incoming | Replies to service commands |
| `thing/product/+/property/set_reply` | Incoming | Replies to dynamic property-set commands (see [Supported Device Models & Dynamic Capabilities](#supported-device-models--dynamic-capabilities)) |
| `thing/product/+/requests` | Incoming | Device-initiated requests |
| `thing/product/+/drc/up` | Incoming | Direct Remote Control upstream data |
| Cloud-to-dock topics | Outgoing | Commands sent to the dock |
| Status reply topics | Outgoing | Topology update acknowledgements |

---

## Port

The adapter listens on port `9001` for incoming gRPC commands from the platform (HTTP and gRPC share the same port via `use-separate-server=false`).
