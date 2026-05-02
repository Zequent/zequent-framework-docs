# Edge SDK (Python) — Models Reference

All public dataclasses are exported from the top-level `edge_sdk` package. They are plain `@dataclass` objects (not Pydantic), so construction is cheap and they serialize to/from protobuf via internal `_converters.py` modules you should not need to touch.

For Java, see [edge-sdk-models.md](edge-sdk-models.md).

---

## Common request / response

### `RequestContext`

```python
@dataclass
class RequestContext:
    tid: str                  # transaction id
    sn: str                   # asset serial number
    timestamp: datetime
    metadata: dict[str, str]  # arbitrary platform-provided headers
```

### `EdgeResponse`

```python
@dataclass
class EdgeResponse:
    tid: str
    sn: str
    success: bool
    message: str
    error: ErrorMessage | None = None
    progress: CommandProgress | None = None
```

Constructors:

| Factory                                           | Result                            |
|---------------------------------------------------|-----------------------------------|
| `EdgeResponse.success(tid, sn, message="")`       | `success=True`                    |
| `EdgeResponse.error(tid, sn, message, code=None)` | `success=False`, fills `error`    |
| `EdgeResponse.not_supported(tid, sn)`             | `success=False`, `code=NOT_IMPLEMENTED` |
| `EdgeResponse.in_progress(tid, sn, progress)`     | `success=True`, fills `progress`  |

### `ErrorMessage`

```python
@dataclass
class ErrorMessage:
    code: ErrorCode
    message: str
    details: dict[str, str]
```

### `CommandProgress`

```python
@dataclass
class CommandProgress:
    percent: int        # 0..100
    status: str         # human-readable phase, e.g. "climbing"
```

### `Coordinates`

```python
@dataclass
class Coordinates:
    latitude: float
    longitude: float
    altitude: float
    heading: float | None = None
```

---

## Asset model

### `Asset`

```python
@dataclass
class Asset:
    sn: str
    name: str
    type: AssetType
    vendor: AssetVendor
    id: str | None = None
    organization_id: str | None = None
    sub_assets: list[SubAsset] = field(default_factory=list)
    metadata: dict[str, str] = field(default_factory=dict)
```

### `SubAsset`

```python
@dataclass
class SubAsset:
    sn: str
    parent_sn: str
    name: str
    id: str | None = None
    type: AssetType | None = None
    vendor: AssetVendor | None = None
    metadata: dict[str, str] = field(default_factory=dict)
```

---

## Telemetry

### `AssetTelemetry`

```python
@dataclass
class AssetTelemetry:
    id: str
    timestamp: datetime
    position: AssetPositionState | None = None
    mode: AssetMode | None = None
    battery_percentage: int | None = None
    environment_temperature: float | None = None
    humidity: float | None = None
    rainfall: Rainfall | None = None
    network: AssetNetworkInfo | None = None
    cover_state: AssetCoverState | None = None
    air_conditioner: AssetAirConditioner | None = None
    sub_assets: list[AssetSubAssetInfo] = field(default_factory=list)
    sensors: list[SensorData] = field(default_factory=list)
```

### `SubAssetTelemetry`

```python
@dataclass
class SubAssetTelemetry:
    id: str
    parent_id: str
    timestamp: datetime
    mode: SubAssetMode | None = None
    battery: SubAssetBatteryInfo | None = None
    position: AssetPositionState | None = None
    payload: PayloadTelemetry | None = None
    network: AssetNetworkInfo | None = None
```

### `PayloadTelemetry`

```python
@dataclass
class PayloadTelemetry:
    camera: CameraData | None = None
    range_finder: RangeFinderData | None = None
    sensors: list[SensorData] = field(default_factory=list)
```

### Helper data classes

- `AssetPositionState(latitude, longitude, altitude, heading=None, vertical_speed=None, horizontal_speed=None)`
- `SubAssetBatteryInfo(percentage, voltage_mv=None, temperature=None, charging=None, cycle_count=None)`
- `AssetNetworkInfo(type: NetworkType, quality: NetworkStateQuality, rssi=None, snr=None)`
- `AssetAirConditioner(mode: AssetAirConditionerState, target_temperature=None, current_temperature=None)`
- `AssetSubAssetInfo(sn, mode, battery_percentage)`
- `CameraData(zoom_factor=None, lens=None, recording=None)`
- `RangeFinderData(distance_m=None, target_lat=None, target_lon=None, target_alt=None)`
- `SensorData(name, unit, value, timestamp=None)`

---

## Tasks and missions

### `Task`

```python
@dataclass
class Task:
    id: str
    mission_id: str | None
    type: TaskType
    status: TaskStatus
    waypoint_config: WaypointTaskConfig | None = None
    detect_config: DetectTaskConfig | None = None
    area_mapping_config: AreaMappingTaskConfig | None = None
    poi_config: PoiTaskConfig | None = None
    follow_config: FollowTaskConfig | None = None
    track_config: TrackTaskConfig | None = None
```

Exactly one `*_config` is populated based on `type`.

### `Mission`

```python
@dataclass
class Mission:
    id: str
    name: str
    type: MissionType
    status: MissionStatus
    tasks: list[Task] = field(default_factory=list)
```

### Task config variants

| Class                     | Key fields                                                                                  |
|---------------------------|---------------------------------------------------------------------------------------------|
| `WaypointTaskConfig`      | `waypoints: list[Waypoint]`, `cruise_speed`, `auto_flight`                                  |
| `DetectTaskConfig`        | `ai_model_id`, `detection_parameters: list[DetectionParameter]`, plus inherited waypoints   |
| `AreaMappingTaskConfig`   | `vertices: list[AreaVertex]`, `survey_altitude`, `overlap_front`, `overlap_side`            |
| `PoiTaskConfig`           | `centre: Coordinates`, `radius_m`, `loops`, `direction`                                     |
| `FollowTaskConfig`        | `subject_sn`, `distance_m`, `bearing_deg`                                                   |
| `TrackTaskConfig`         | `class_name`, `confidence_min`, `loss_timeout_sec`                                          |

### `Waypoint` and `AreaVertex`

```python
@dataclass
class Waypoint:
    latitude: float
    longitude: float
    altitude: float
    heading: float | None = None
    speed: float | None = None
    actions: list[str] = field(default_factory=list)

@dataclass
class AreaVertex:
    latitude: float
    longitude: float
    altitude: float | None = None
```

---

## Detection

```python
@dataclass
class DetectionResult:
    class_name: str
    confidence: float
    box: BoundingBox

@dataclass
class BoundingBox:
    x: int
    y: int
    width: int
    height: int

@dataclass
class DetectionResponse:
    task_id: str
    timestamp: datetime
    results: list[DetectionResult]
```

---

## Capabilities

```python
@dataclass
class Capability:
    command: str       # e.g. "TakeOff", "GoTo"
    description: str
    available: bool

@dataclass
class Capabilities:
    asset_sn: str
    asset_type: AssetType
    capabilities: list[Capability]
    timestamp: datetime
```

Use `EdgeAdapter._auto_capabilities(sn, asset_type)` to construct one based on which methods you've overridden.

---

## Enums

`IntEnum` types from `edge_sdk.models.common`:

| Enum                          | Values (selected)                                                            |
|-------------------------------|------------------------------------------------------------------------------|
| `AssetType`                   | `DOCK`, `DRONE`, `VEHICLE`, ...                                              |
| `AssetVendor`                 | `DJI`, `AUTEL`, `PARROT`, `CUSTOM`                                           |
| `AssetConnection`             | `ONLINE`, `OFFLINE`                                                          |
| `LiveStreamType`              | `RTMP`, `WEBRTC`, `HLS`                                                      |
| `AssetMode`                   | `IDLE`, `WORKING`, `MAINTENANCE`, ...                                        |
| `SubAssetMode`                | `DOCKED`, `FLYING`, `RETURNING`, ...                                         |
| `ManualControlState`          | `IDLE`, `ACTIVE`                                                             |
| `AssetCoverState`             | `OPEN`, `CLOSED`, `OPENING`, `CLOSING`                                       |
| `AssetAirConditionerState`    | `OFF`, `COOLING`, `HEATING`, `AUTO`                                          |
| `TaskType`                    | `WAYPOINT`, `DETECT`, `AREA_MAPPING`, `POI`, `FOLLOW`, `TRACK`               |
| `TaskStatus`                  | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`                     |
| `MissionType`                 | `IMMEDIATE`, `SCHEDULED`                                                     |
| `MissionStatus`               | `DRAFT`, `ACTIVE`, `COMPLETED`, `CANCELLED`                                  |
| `ErrorCode`                   | `NOT_IMPLEMENTED`, `HARDWARE_ERROR`, `INVALID_REQUEST`, ...                  |
| `Rainfall`                    | `NONE`, `LIGHT`, `MODERATE`, `HEAVY`                                         |
| `NetworkType`                 | `LTE`, `5G`, `WIFI`, `ETHERNET`                                              |
| `NetworkStateQuality`         | `WEAK`, `FAIR`, `STRONG`, `EXCELLENT`                                        |

Pass them as actual enum members, not raw ints — the proto converters accept both, but explicit enums make your code self-documenting.
