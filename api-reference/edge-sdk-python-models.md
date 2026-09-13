# Edge SDK (Python) — Models Reference

All public dataclasses are exported from the top-level `edge_sdk` package. They are plain `@dataclass` objects (not Pydantic), so construction is cheap and they serialize to/from protobuf via internal `_converters.py` modules you should not need to touch.

For Java, see [edge-sdk-models.md](edge-sdk-models.md).

---

## Common request / response

### `RequestContext`

```python
@dataclass
class RequestContext:
    tid: str        # transaction id
    sn: str         # asset serial number the command is addressed to
    timestamp: datetime
```

### `EdgeResponse`

```python
@dataclass
class EdgeResponse:
    tid: str
    sn: str
    success: bool
    asset_id: str | None = None
    message: str | None = None
    error: ErrorMessage | None = None
    progress: CommandProgress | None = None
    stream_url: str | None = None   # populated only for StartLiveStream responses
    video_id: str | None = None
    external_execution_id: str | None = None  # set when this command keeps running asynchronously
```

Constructors — note these are named `ok`/`fail`, not `success`/`error`:

| Factory                                                                    | Result                                   |
|-----------------------------------------------------------------------------|-------------------------------------------|
| `EdgeResponse.ok(tid, sn, message=None, *, external_execution_id=None, ...)` | `success=True`                             |
| `EdgeResponse.fail(tid, sn, error: ErrorMessage, asset_id=None)`            | `success=False`, fills `error`             |
| `EdgeResponse.not_supported(tid, sn)`                                       | `success=False`, error code `SDK_ERROR` |

```python
return EdgeResponse.ok(ctx.tid, ctx.sn, "Takeoff initiated")

return EdgeResponse.fail(ctx.tid, ctx.sn, ErrorMessage(message="Device busy", code=ErrorCode.HARDWARE_ERROR))
```

Set `external_execution_id` on `ok(...)` when the command you just accepted keeps running asynchronously — use your own vendor execution id if you have one (e.g. a DJI `flightId`); otherwise leave it unset and the platform falls back to its own correlation id.

### `ErrorMessage`

```python
@dataclass
class ErrorMessage:
    message: str
    code: ErrorCode
    timestamp: datetime | None = None
```

### `CommandProgress`

```python
@dataclass
class CommandProgress:
    progress: float          # 0.0-100.0
    state: str
    left_time_seconds: float
```

### `Coordinates`

```python
@dataclass
class Coordinates:
    latitude: float
    longitude: float
    altitude: float
```

---

## Asset model

### `Asset`

```python
@dataclass
class Asset:
    id: str | None
    sn: str
    name: str
    type: AssetType
    vendor: AssetVendor
    connection: AssetConnection
    model: str
    organization: str
    system_connection_string: str | None = None
    external_device_type: str | None = None
    external_device_sub_type: str | None = None
    external_id: str | None = None
    live_stream_push_url: str | None = None
    live_stream_pull_url: str | None = None
    created_at: datetime | None = None
```

### `SubAsset`

```python
@dataclass
class SubAsset:
    id: str | None
    sn: str
    name: str
    type: AssetType
    vendor: AssetVendor
    connection: AssetConnection
    model: str
    system_connection_string: str | None = None
    external_device_type: str | None = None
    external_device_sub_type: str | None = None
    external_id: str | None = None
    stream_url_predefined: bool | None = None
    live_stream_push_url: str | None = None
    live_stream_pull_url: str | None = None
    created_at: datetime | None = None
```

`SubAsset` has no `parent_sn` field — the parent/sub-asset relationship is established through the Connector Service's registration call, not carried on the dataclass itself.

---

## Telemetry

`AssetTelemetry` and `SubAssetTelemetry` are flat — position and movement fields (`latitude`, `longitude`, `absolute_altitude`, `relative_altitude`, ...) live directly on them, not nested in a separate position object.

### `AssetTelemetry`

```python
@dataclass
class AssetTelemetry:
    id: str
    timestamp: datetime | None = None
    latitude: float | None = None
    longitude: float | None = None
    absolute_altitude: float | None = None
    relative_altitude: float | None = None
    environment_temp: float | None = None
    inside_temp: float | None = None
    humidity: float | None = None
    mode: AssetMode | None = None
    rainfall: Rainfall | None = None
    sub_asset_info: AssetSubAssetInfo | None = None
    sub_asset_at_home: bool | None = None
    sub_asset_charging: bool | None = None
    sub_asset_percentage: float | None = None
    heading: float | None = None
    debug_mode_open: bool | None = None
    has_active_manual_control_session: bool | None = None
    cover_state: AssetCoverState | None = None
    working_voltage: int | None = None
    working_current: int | None = None
    supply_voltage: int | None = None
    wind_speed: float | None = None
    position_valid: bool | None = None
    network_info: AssetNetworkInfo | None = None
    air_conditioner: AssetAirConditioner | None = None
    manual_control_state: ManualControlState | None = None
    position_state: AssetPositionState | None = None
```

### `SubAssetTelemetry`

```python
@dataclass
class SubAssetTelemetry:
    id: str
    timestamp: datetime | None = None
    latitude: float | None = None
    longitude: float | None = None
    absolute_altitude: float | None = None
    relative_altitude: float | None = None
    horizontal_speed: float | None = None
    vertical_speed: float | None = None
    wind_speed: float | None = None
    wind_direction: str | None = None
    heading: float | None = None
    gear: int | None = None
    payload: PayloadTelemetry | None = None
    battery: SubAssetBatteryInfo | None = None
    height_limit: int | None = None
    home_distance: float | None = None
    total_movement_distance: float | None = None
    total_movement_time: float | None = None
    mode: SubAssetMode | None = None
    country: str | None = None
```

### `PayloadTelemetry`

```python
@dataclass
class PayloadTelemetry:
    id: str
    name: str
    timestamp: datetime | None = None
    camera: CameraData | None = None
    range_finder: RangeFinderData | None = None
    sensor: SensorData | None = None
```

### Helper data classes

- `AssetPositionState(gps_number=None, rtk_number=None, quality=None)` — GNSS fix quality, not a coordinate
- `SubAssetBatteryInfo(percentage=None, remaining_time=None, return_to_home_power=None)`
- `AssetNetworkInfo(type: NetworkType, rate=None, quality: NetworkStateQuality)`
- `AssetAirConditioner(state: AssetAirConditionerState, switch_time=None)`
- `AssetSubAssetInfo(sn=None, model=None, paired=None, online=None)`
- `CameraData(current_lens=None, gimbal_pitch=None, gimbal_yaw=None, zoom_factor=None, gimbal_roll=None)`
- `RangeFinderData(target_latitude=None, target_longitude=None, target_distance=None, target_altitude=None)`
- `SensorData(target_temperature=None)`

---

## Tasks and missions

Most edge adapters never construct these directly — the platform drives task execution by passing your adapter a `task_id: str`, and you fetch whatever detail you need through `ConnectorClient` (see [Mission Autonomy](../edge-sdk/edge-sdk-python-mission-autonomy.md)). These dataclasses exist if you need the richer, vendor-specific task shape (e.g. mirroring a DJI wayline mission).

### `Task`

```python
@dataclass
class Task:
    status: TaskStatus
    id: str | None = None
    mission_id: str | None = None
    name: str | None = None
    task_type: TaskType | None = None
    asset_id: str | None = None
    sn_number: str | None = None
    current_progress: int | None = None
    current_step: str | None = None
    waypoint_config: WaypointTaskConfig | None = None
    detect_config: DetectTaskConfig | None = None
    area_mapping_config: AreaMappingTaskConfig | None = None
    poi_config: PoiTaskConfig | None = None
    follow_config: FollowTaskConfig | None = None
    track_config: TrackTaskConfig | None = None
    created_at: datetime | None = None
    modified_at: datetime | None = None
    # additional fields: description, config, break_reason, external_command_type, modified_from
```

At most one `*_config` is populated based on `task_type`.

### `Mission`

```python
@dataclass
class Mission:
    name: str
    description: str
    status: MissionStatus
    type: MissionType
    id: str | None = None
    tasks: list[Task] = field(default_factory=list)
    assigned_assets: list[str] = field(default_factory=list)
    start_date: datetime | None = None
    end_date: datetime | None = None
```

### Task config variants (representative fields — each has 10-25 fields; see the SDK source for the full list)

| Class                     | Representative fields                                                                        |
|---------------------------|-----------------------------------------------------------------------------------------------|
| `WaypointTaskConfig`      | `flight_id`, `waypoints: list[Waypoint]`, `global_speed`, `rth_altitude`, `rth_mode`          |
| `DetectTaskConfig`        | `ai_model_id`, `detection_targets`, `min_confidence`, `detection_parameters: list[DetectionParameter]` |
| `AreaMappingTaskConfig`   | `survey_altitude`, `area_vertices: list[AreaVertex]`, `front_overlap`, `side_overlap`, `flight_pattern` |
| `PoiTaskConfig`           | `poi_latitude`, `poi_longitude`, `poi_altitude`, `orbit_radius`, `orbit_speed`, `number_of_orbits` |
| `FollowTaskConfig`        | `target_type`, `follow_distance`, `follow_mode`, `max_speed`, `lost_target_action`            |
| `TrackTaskConfig`         | `target_type`, `tracking_mode`, `confidence_threshold`, `max_movement_radius`, `lost_target_action` |

### `Waypoint` and `AreaVertex`

```python
@dataclass
class Waypoint:
    latitude: float
    longitude: float
    altitude: float | None = None
    speed: float | None = None
    fly_through: bool | None = None
    vehicle_action: VehicleAction | None = None
    wp_order: int | None = None
    gimbal_pitch: int | None = None

@dataclass
class AreaVertex:
    latitude: float
    longitude: float
    order: int | None = None
```

---

## Detection

```python
@dataclass
class BoundingBox:
    x: float
    y: float
    width: float
    height: float

@dataclass
class DetectionResult:
    object_id: str
    object_type: str
    confidence: float
    bounding_box: BoundingBox

@dataclass
class DetectionResponse:
    detections: list[DetectionResult] = field(default_factory=list)

@dataclass
class DetectionBatch:
    """Batch of detection results published via DetectionPublisher/LiveDataService.produce_detection."""
    sn: str = ""
    detections: list[DetectionResult] = field(default_factory=list)
    stream_url: str | None = None
```

---

## Notifications

```python
@dataclass
class AssetStatusEvent:
    sn: str
    online: bool
    asset_id: str | None = None
    message: str | None = None

@dataclass
class MissionEvent:
    mission_id: str
    mission_type: MissionType
    status: MissionStatus
    sn: str = ""
    message: str | None = None

@dataclass
class TaskEvent:
    """Task lifecycle event."""
    task_id: str
    task_type: TaskType
    status: TaskStatus
    sn: str = ""  # asset serial number - required for multi-asset adapters
    progress: float | None = None
    message: str | None = None
    external_task_type: str | None = None
```

See [Live Data — Notifications](../edge-sdk/edge-sdk-python-live-data.md#notifications).

---

## Capabilities

```python
@dataclass
class Capability:
    command: str       # e.g. "TakeOff", "GoTo"
    description: str
    available: bool
    unavailable_reason: str | None = None
    metadata: dict[str, str] = field(default_factory=dict)

@dataclass
class Capabilities:
    asset_sn: str
    asset_type: AssetType
    capabilities: list[Capability] = field(default_factory=list)
    timestamp: datetime | None = None
```

Use `EdgeAdapter._auto_capabilities(sn, asset_type)` to construct one based on which methods you've overridden.

---

## Enums

`IntEnum` types from `edge_sdk.models.common`:

| Enum                          | Values                                                                                          |
|-------------------------------|---------------------------------------------------------------------------------------------------|
| `AssetType`                   | `UNKNOWN`, `AIRCRAFT`, `DOCK`, `SENSOR`, `CAMERA`, `OTHER`, `JAMMER`, `CYBER_ATTACK`, `SAPIENT`    |
| `AssetVendor`                 | `DJI`, `AUTEL`, `ROS`, `MAVLINK`, `RTMP_RTSP`, `SAPIENT`, `BETAFLIGHT`                             |
| `AssetConnection`              | `MQTT`, `TCP`, `SERIAL`                                                                            |
| `LiveStreamType`              | `UNKNOWN`, `RTMP`, `RTSP`, `WEBRTC`                                                                |
| `AssetMode`                   | `IDLE`, `DEBUGGING`, `REMOTE_DEBUGGING`, `UPGRADING`, `WORKING`, `TO_BE_CALIBRATED`, `OFFLINE`     |
| `SubAssetMode`                | `IDLE`, `TAKEOFF_PREPARE`, `TAKEOFF_FINISHED`, `MANUAL`, `TAKEOFF_AUTO`, `WAYLINE`, `PANORAMIC_SHOT`, `ACTIVE_TRACK`, `ADS_B_AVOIDANCE`, `RETURN_AUTO`, `LANDING_AUTO`, `LANDING_FORCE`, and more (mirrors DJI's flight-mode set) |
| `ManualControlState`          | `DISCONNECTED`, `CONNECTING`, `CONNECTED`                                                          |
| `AssetCoverState`             | `CLOSED`, `OPENED`, `HALF_OPEN`, `ABNORMAL`                                                        |
| `AssetAirConditionerState`    | `IDLE`, `COOL`, `HEAT`, `DEHUMIDIFICATION`, plus `*_EXIT`/`*_PREPARATION` transition states        |
| `TaskType`                    | `UNSPECIFIED`, `DETECT`, `AREA_MAPPING`, `WAYPOINT`, `POI`, `FOLLOW`, `TRACK`, `COUNTER_DRONE`      |
| `TaskStatus`                  | `UNKNOWN`, `DRAFT`, `SCHEDULED`, `RUNNING`, `ERROR`, `COMPLETED`, `PREPARED`, `PAUSED`              |
| `MissionType`                 | `STANDARD`, `REMOTE_OPS`, `DRF`, `MISSION`                                                         |
| `MissionStatus`               | `UNKNOWN`, `DRAFT`, `ACTIVE`, `INACTIVE`, `ERROR`                                                  |
| `TaskStatus`                  | `UNKNOWN`, `DRAFT`, `SCHEDULED`, `RUNNING`, `ERROR`, `COMPLETED`, `PREPARED`, `PAUSED`              |
| `ErrorCode`                   | `SYSTEM_ERROR`, `CLIENT_ERROR`, `SDK_ERROR`, `SERVICE_ERROR`, `ASSET_ERROR`                         |
| `Rainfall`                    | `NO`, `LIGHT`, `MODERATE`, `HEAVY`                                                                 |
| `NetworkType`                 | `NETWORK_4G`, `ETHERNET`                                                                            |
| `NetworkStateQuality`         | `NO_SIGNAL`, `BAD`, `POOR`, `FAIR`, `GOOD`, `EXCELLENT`                                             |

Pass them as actual enum members, not raw ints — the proto converters accept both, but explicit enums make your code self-documenting.
