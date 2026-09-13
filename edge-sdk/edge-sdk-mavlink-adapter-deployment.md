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
| `SendCustomCommand("mission.waypoint.execute")` | uploads the inline waypoint list as a MAVSDK mission (`mission.upload_mission`), arms, then `mission.start_mission` |
| `StopTask` | `mission.pause_mission` |
| `RegisterAsset` | registers with the Connector Service and eagerly opens the MAVSDK connection |
| `DeRegisterAsset` | releases the MAVSDK connection |

Every other command defaults to "not supported" and is reported as such through `GetCapabilities` —
**including `PrepareTask`/`StartTask`.** Mission/Task CRUD is retired platform-wide, so there's no
RPC left to resolve a bare task ID into waypoint data; this adapter takes the command-based path
instead, the same way the simulator and edge-dji do. See
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md) for the full picture across adapters.

---

## Environment Variables

All of these except `MAVLINK_CONNECTION`/`MAVLINK_PRECONNECT_SN` come from the shared Python Edge
SDK's `EdgeAdapterConfig.from_env()` — this adapter passes no overrides, so these are that class's
own real defaults, confirmed in `edge_sdk/config.py`:

| Variable | Default | Purpose |
| --- | --- | --- |
| `GRPC_HOST` | `0.0.0.0` | gRPC server bind host |
| `GRPC_PORT` | `50051` | gRPC server bind port — this is what the platform reaches the adapter on |
| `CONNECTOR_HOST` | `localhost` | Connector Service host |
| `CONNECTOR_PORT` | `50053` | Connector Service port — **not** the real Connector Service platform port (`8010`, see the [image table](../README.md#platform-service-images)); set it explicitly, don't rely on this default |
| `TELEMETRY_HOST` | `localhost` | Live Data Service host — telemetry forwarding is always on, there's no way to disable it via this variable |
| `TELEMETRY_PORT` | `50052` | Live Data Service port — likewise not the real platform port (`8003`); set it explicitly |
| `ADAPTER_SN` | `mavlink-adapter` | Serial number used on outgoing telemetry frames |
| `EDGE_ENDPOINT` | _unset_ | Endpoint this adapter advertises via optional Redis service-discovery registration |
| `ASSET_TYPE` | _unset_ | Full proto-style name, **not** the bare enum member — `ASSET_TYPE_AIRCRAFT`, not `AIRCRAFT`. A bare value fails `proto_enum_lookup` and silently skips registration (logged as a warning only) |
| `ASSET_VENDOR` | _unset_ | Same requirement — `ASSET_VENDOR_MAVLINK`, not `MAVLINK` |
| `REDIS_URL` | `redis://localhost:6379` | Two independent uses: the SDK's own service-discovery registration (only when `EDGE_ENDPOINT`/`ASSET_TYPE`/`ASSET_VENDOR` are all set), and this adapter's own asset online/offline caching (used whenever the variable is set, regardless of `EDGE_ENDPOINT`) |
| `LOG_LEVEL` | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `LOG_FORMAT` | `json` | `json` or `text` |

`MISSION_AUTONOMY_HOST`/`PORT` are read by the shared SDK config but have no effect here — this adapter doesn't talk to the Mission Autonomy Service directly (see [Edge SDK — Mission Autonomy](edge-sdk-python-mission-autonomy.md)).

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
ASSET_TYPE=ASSET_TYPE_AIRCRAFT
ASSET_VENDOR=ASSET_VENDOR_MAVLINK
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
              value: "ASSET_TYPE_AIRCRAFT"
            - name: ASSET_VENDOR
              value: "ASSET_VENDOR_MAVLINK"
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
