# MAVLink Edge Adapter -- Deployment Guide

The MAVLink Edge Adapter connects PX4 and ArduPilot vehicles to the Zequent platform. It's built on the Python Edge SDK and [MAVSDK-Python](https://github.com/mavlink/MAVSDK-Python), exposing the standard `EdgeAdapterService` gRPC interface and translating incoming platform commands into MAVSDK calls against the vehicle identified by the request's serial number.

---

## Prerequisites

- Access to the Zequent container registry (`ghcr.io/zequent`)
- A reachable MAVLink endpoint for your vehicle (serial, UDP, or TCP — anything MAVSDK supports)
- Running Zequent platform services: Connector Service, Live Data Service

---

## Implemented Commands

| Command | Maps to |
| --- | --- |
| `TakeOff` | `set_takeoff_altitude` → `arm` → `takeoff` |
| `GoTo` | `action.goto_location(lat, lon, alt, yaw=0)` |
| `ReturnToHome` | `action.return_to_launch` (with optional RTL altitude) |
| `RebootAsset` | `action.reboot` |
| `EnterManualControl` | `manual_control.start_position_control` |
| `ExitManualControl` | `action.hold` |
| `ManualControlInput` | streams stick input → `manual_control.set_manual_control_input` |
| `PrepareTask` | fetches the task and uploads it as a MAVSDK mission (`mission.upload_mission`) |
| `StartTask` | `mission.start_mission` |
| `StopTask` | `mission.pause_mission` |
| `RegisterAsset` | registers with the Connector Service and eagerly opens the MAVSDK connection |
| `DeRegisterAsset` | releases the MAVSDK connection |

Every other command defaults to "not supported" and is reported as such through `GetCapabilities`.

---

## Environment Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `GRPC_HOST` | `[::]` | gRPC server bind host |
| `GRPC_PORT` | `50051` | gRPC server bind port — this is what the platform reaches the adapter on |
| `ZQNT_CLAIM_CODE` | _unset_ | A one-time pairing code from the console's **Pair device** dialog. Used only when the platform does not already know a serial number this adapter is bringing up: the code decides which organization the resulting asset belongs to, and that cannot be changed afterwards. Leave it unset once the assets exist — an already-paired device does not need it, and the code is single-use. Without it, an unknown serial simply has no asset, and the adapter creates nothing. |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `8010` | Connector Service port |
| `TELEMETRY_HOST` | _unset_ | Live Data Service host — telemetry forwarding is disabled while this is unset |
| `TELEMETRY_PORT` | `50052` | Live Data Service port — override to `8003` to match a real deployment |
| `ADAPTER_SN` | `mavlink-adapter` | Serial number used on outgoing telemetry frames |
| `EDGE_ENDPOINT` | _unset_ | Endpoint this adapter advertises via optional Redis service-discovery registration |
| `ASSET_TYPE` | _unset_ | Proto-style `AssetType` name — typically `AIRCRAFT` |
| `ASSET_VENDOR` | _unset_ | Proto-style `AssetVendor` name — typically `MAVLINK` |
| `REDIS_URL` | `redis://localhost:6379` | Used only when `EDGE_ENDPOINT` is set |
| `LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `LOG_FORMAT` | `json` | `json` or `text` |

There is no `MISSION_AUTONOMY_HOST`/`PORT` — this adapter doesn't talk to the Mission Autonomy Service directly (see [Edge SDK — Mission Autonomy](edge-sdk-python-mission-autonomy.md)).

The vehicle's MAVLink connection string (e.g. `udp://:14540`, `serial:///dev/ttyUSB0:57600`) is resolved per-asset at registration time, not from a single global environment variable — see the adapter's own README for the exact resolution order if you need to override it.

---

## Docker Compose

```yaml
services:
  edge-mavlink:
    image: ghcr.io/zequent/zqnt-adapter-mavlink:1.3.0
    env_file:
      - .env
    ports:
      - "9001:50051"
    restart: unless-stopped
```

`.env`:

```bash
GRPC_PORT=50051
CONNECTOR_HOST=connector-service
CONNECTOR_PORT=8010
TELEMETRY_HOST=live-data-service
TELEMETRY_PORT=8003
ADAPTER_SN=mavlink-01
ASSET_TYPE=AIRCRAFT
ASSET_VENDOR=MAVLINK
```

---

## Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-mavlink
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edge-mavlink
  template:
    metadata:
      labels:
        app: edge-mavlink
    spec:
      containers:
        - name: edge-mavlink
          image: ghcr.io/zequent/zqnt-adapter-mavlink:1.3.0
          ports:
            - containerPort: 50051
          env:
            - name: CONNECTOR_HOST
              value: "connector-service"
            - name: CONNECTOR_PORT
              value: "8010"
            - name: TELEMETRY_HOST
              value: "live-data-service"
            - name: TELEMETRY_PORT
              value: "8003"
            - name: ADAPTER_SN
              value: "mavlink-01"
            - name: ASSET_TYPE
              value: "AIRCRAFT"
            - name: ASSET_VENDOR
              value: "MAVLINK"
---
apiVersion: v1
kind: Service
metadata:
  name: edge-mavlink
spec:
  selector:
    app: edge-mavlink
  ports:
    - port: 50051
      targetPort: 50051
```

---

## Testing against a simulator

The adapter's own test setup uses headless PX4 SITL over UDP — useful for verifying your deployment before connecting real hardware:

```bash
docker run --rm -it --name px4-sim --net=host \
  jonasvautherin/px4-gazebo-headless:1.14.0 <HOST_IP>
```

Replace `<HOST_IP>` with the IP address of the host running the adapter — PX4 streams MAVLink telemetry back to that address. The adapter falls back to `udp://:14540` when no connection string is registered for the asset, which matches this simulator's default.

---

## Port

The adapter listens on `50051` for incoming gRPC commands from the platform.
