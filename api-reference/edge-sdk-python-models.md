# Edge SDK (Python) — Models Reference

> For 1.3.x (end of life), see the [1.3 Models reference](edge-sdk-python-models-1.3.md).

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
| `EdgeResponse.ok(tid, sn, message=None, ..., external_execution_id=None)` | `success=True`                             |
| `EdgeResponse.fail(tid, sn, error: ErrorMessage, asset_id=None)`            | `success=False`, fills `error`             |
| `EdgeResponse.not_supported(tid, sn)`                                       | `success=False`, error code `SDK_ERROR` |

```python
return EdgeResponse.ok(ctx.tid, ctx.sn, "Takeoff initiated")

return EdgeResponse.fail(ctx.tid, ctx.sn, ErrorMessage(message="Device busy", code=ErrorCode.ASSET_ERROR))
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
class CommandExecutionEvent:
    """The outcome of a command you accepted."""
    external_execution_id: str
    status: CommandExecutionStatus
    sn: str = ""                      # the asset that ran it - always set it
    command_id: str | None = None
    progress: float | None = None     # 0.0 - 1.0, while RUNNING
    message: str | None = None
    output: dict | None = None        # result on SUCCEEDED
    occurred_at: datetime | None = None  # defaults to publish time
```

See [Live Data — Notifications](../edge-sdk/edge-sdk-python-live-data.md#notifications).

---

## Capabilities

```python
@dataclass
class Capability:
    command_id: str                   # e.g. "flight.takeoff", "mission.waypoint.execute"
    description: str = ""
    state: CapabilityState = CapabilityState.AVAILABLE
    display_name: str | None = None
    unavailable_reason: str | None = None
    metadata: dict[str, str] = field(default_factory=dict)
    input_schema: dict | None = None  # JSON Schema of the parameters
    output_schema: dict | None = None
    target: CapabilityTarget | None = None   # type (ASSET, SUB_ASSET, PAYLOAD, COMPONENT) + target_ref
    schema_version: str | None = None
    skill_id: str | None = None
    source: CapabilitySource = CapabilitySource.EDGE_ADAPTER

@dataclass
class Capabilities:
    asset_sn: str
    asset_type: AssetType
    capabilities: list[Capability] = field(default_factory=list)
    timestamp: datetime | None = None
```

Build it with `EdgeAdapter._auto_capabilities(sn, asset_type)`, and declare extra commands with
`register_command` — see the [Edge Adapter reference](edge-sdk-python-adapter-reference.md#capabilities-and-command-registration).

---

## Enums

`IntEnum` types from `edge_sdk.models.common`:

| Enum                          | Values                                                                                          |
|-------------------------------|---------------------------------------------------------------------------------------------------|
| `AssetType`                   | `UNKNOWN`, `AIRCRAFT`, `DOCK`, `SENSOR`, `CAMERA`, `OTHER`, `JAMMER`, `CYBER_ATTACK`, `SAPIENT`, `RNS` |
| `AssetVendor`                 | `DJI`, `AUTEL`, `ROS`, `MAVLINK`, `RTMP_RTSP`, `SAPIENT`, `BETAFLIGHT`, `RNS`, `ZQNT`, `SIMULATOR` |
| `AssetConnection`              | `MQTT`, `TCP`, `SERIAL`                                                                            |
| `LiveStreamType`              | `UNKNOWN`, `RTMP`, `RTSP`, `WEBRTC`                                                                |
| `AssetMode`                   | `IDLE`, `DEBUGGING`, `REMOTE_DEBUGGING`, `UPGRADING`, `WORKING`, `TO_BE_CALIBRATED`, `OFFLINE`     |
| `SubAssetMode`                | `IDLE`, `TAKEOFF_PREPARE`, `TAKEOFF_FINISHED`, `MANUAL`, `TAKEOFF_AUTO`, `WAYLINE`, `PANORAMIC_SHOT`, `ACTIVE_TRACK`, `ADS_B_AVOIDANCE`, `RETURN_AUTO`, `LANDING_AUTO`, `LANDING_FORCE`, and more (mirrors DJI's flight-mode set) |
| `ManualControlState`          | `DISCONNECTED`, `CONNECTING`, `CONNECTED`                                                          |
| `AssetCoverState`             | `CLOSED`, `OPENED`, `HALF_OPEN`, `ABNORMAL`                                                        |
| `AssetAirConditionerState`    | `IDLE`, `COOL`, `HEAT`, `DEHUMIDIFICATION`, plus `*_EXIT`/`*_PREPARATION` transition states        |
| `MissionType`                 | `STANDARD`, `REMOTE_OPS`, `DRF`, `MISSION`                                                         |
| `MissionStatus`               | `UNKNOWN`, `DRAFT`, `ACTIVE`, `INACTIVE`, `ERROR`                                                  |
| `CommandExecutionStatus`      | `UNSPECIFIED`, `ACCEPTED`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED`                            |
| `CapabilityState`             | `UNSPECIFIED`, `AVAILABLE`, `TEMPORARILY_UNAVAILABLE`, `UNSUPPORTED`, `REQUIRES_AUTHORIZATION`      |
| `CapabilityTargetType`        | `UNSPECIFIED`, `ASSET`, `SUB_ASSET`, `PAYLOAD`, `COMPONENT`                                         |
| `ErrorCode`                   | `SYSTEM_ERROR`, `CLIENT_ERROR`, `SDK_ERROR`, `SERVICE_ERROR`, `ASSET_ERROR`                         |
| `Rainfall`                    | `NO`, `LIGHT`, `MODERATE`, `HEAVY`                                                                 |
| `NetworkType`                 | `NETWORK_4G`, `ETHERNET`                                                                            |
| `NetworkStateQuality`         | `NO_SIGNAL`, `BAD`, `POOR`, `FAIR`, `GOOD`, `EXCELLENT`                                             |

Pass them as actual enum members, not raw ints — the proto converters accept both, but explicit enums make your code self-documenting.
