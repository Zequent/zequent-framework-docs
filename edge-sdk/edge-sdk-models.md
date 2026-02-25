# Edge SDK -- Models Reference

This document provides a comprehensive reference for all request, response, and data models used in the Zequent Edge SDK.

## Table of Contents

- [Command Models](#command-models)
  - [CommandResult](#commandresult)
  - [CurrentCapabilities](#currentcapabilities)
  - [Capability](#capability)
- [Flight Control Models](#flight-control-models)
  - [TakeOffRequest](#takeoffrequest)
  - [ReturnToHomeRequest](#returntohomerequest)
  - [GoToRequest](#gotorequest)
  - [Coordinates](#coordinates)
- [Manual Control Models](#manual-control-models)
  - [ManualControlRequest](#manualcontrolrequest)
  - [ManualControlInput](#manualcontrolinput)
- [Camera and Gimbal Models](#camera-and-gimbal-models)
  - [LookAtRequest](#lookatrequest)
  - [ChangeLensRequest](#changelensrequest)
  - [ChangeZoomRequest](#changezoomrequest)
- [Live Streaming Models](#live-streaming-models)
  - [LiveStreamStartRequest](#livestreamstartrequest)
  - [LiveStreamStopRequest](#livestreamstoprequest)
- [Telemetry Models](#telemetry-models)
  - [TelemetryRequestData](#telemetryrequestdata)
  - [AssetTelemetryData](#assettelemetrydata)
  - [SubAssetTelemetryData](#subassettelemetrydata)
- [Mission Models](#mission-models)
  - [MissionData](#missiondata)
- [Configuration Models](#configuration-models)
  - [EdgeClientConfig](#edgeclientconfig)

---

## Command Models

### CommandResult

The universal return type for all `EdgeAdapterService` commands.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `success` | `boolean` | Whether the command succeeded |
| `message` | `String` | Human-readable result message |
| `tid` | `String` | Transaction ID for tracing |
| `sn` | `String` | Device serial number |
| `resultType` | `CommandResultType` | Result classification |

**CommandResultType enum:**

| Value | Description |
|-------|-------------|
| `SUCCESS` | Command executed successfully |
| `ERROR` | Command failed |
| `NOT_IMPLEMENTED` | Command is not supported by this adapter |

**Factory methods:**

```java
// Success
CommandResult.success("Message", sn)
CommandResult.success("Message", tid, sn)

// Error
CommandResult.error("Error message", sn)
CommandResult.error("Error message", tid, sn)

// Not implemented (used internally by default methods)
CommandResult.notImplemented("Not supported", sn)
```

**Utility methods:**

```java
boolean isNotImplemented()  // returns true if resultType == NOT_IMPLEMENTED
```

---

### CurrentCapabilities

Reports the set of capabilities that an adapter currently supports.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `assetType` | `AssetTypeEnum` | Type of asset |
| `capabilities` | `Set<Capability>` | Set of supported capabilities |
| `timestamp` | `long` | Timestamp when capabilities were reported |

**Factory methods:**

```java
// Empty capabilities
CurrentCapabilities.empty(sn)

// With specific capabilities
CurrentCapabilities.of(sn, AssetTypeEnum.ASSET_TYPE_DOCK, capabilitySet)
```

---

### Capability

Describes a single capability of the adapter.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `command` | `String` | Command name (e.g., "takeOff", "openCover") |
| `description` | `String` | Human-readable description |
| `available` | `Boolean` | Whether the capability is currently available |
| `unavailableReason` | `String` | Reason if not available (null if available) |
| `metadata` | `Map<String, String>` | Additional metadata key-value pairs |

---

## Flight Control Models

### TakeOffRequest

Request to initiate takeoff.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `coordinates` | `Coordinates` | Takeoff position (latitude, longitude, altitude) |

---

### ReturnToHomeRequest

Request to return the sub-asset (drone) to its home position.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `tid` | `String` | Transaction identifier |
| `sn` | `String` | Device serial number |
| `altitude` | `Float` | Return altitude in meters |

---

### GoToRequest

Request to navigate to specific coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `coordinates` | `Coordinates` | Target position |

---

### Coordinates

Geographic coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `latitude` | `Float` | GPS latitude in degrees |
| `longitude` | `Float` | GPS longitude in degrees |
| `altitude` | `Float` | Altitude in meters |

---

## Manual Control Models

### ManualControlRequest

Request to enter or exit manual control mode.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `enable` | `boolean` | `true` to enter manual control, `false` to exit |

---

### ManualControlInput

A single frame of joystick/stick input for controlling the device in real-time.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `roll` | `Float` | Roll axis value (typically -1.0 to 1.0) |
| `pitch` | `Float` | Pitch axis value (typically -1.0 to 1.0) |
| `yaw` | `Float` | Yaw axis value (typically -1.0 to 1.0) |
| `throttle` | `Float` | Throttle value (typically -1.0 to 1.0) |
| `gimbalPitch` | `Float` | Gimbal pitch adjustment |

This model supports the `@Builder` pattern:

```java
ManualControlInput input = ManualControlInput.builder()
    .sn("DEVICE_SN")
    .roll(0.0f)
    .pitch(0.5f)
    .yaw(0.0f)
    .throttle(0.3f)
    .gimbalPitch(-0.2f)
    .build();
```

---

## Camera and Gimbal Models

### LookAtRequest

Request to point the camera at specific coordinates.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `latitude` | `Double` | Target GPS latitude |
| `longitude` | `Double` | Target GPS longitude |
| `altitude` | `Float` | Target altitude in meters |
| `locked` | `Boolean` | Whether to lock the gimbal on the target |
| `payloadIndex` | `String` | Camera/payload index identifier |

---

### ChangeLensRequest

Request to switch the active camera lens.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `lens` | `String` | Target lens identifier (e.g., "wide", "zoom", "ir") |
| `videoId` | `String` | Video stream identifier |

---

### ChangeZoomRequest

Request to adjust the camera zoom level.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `lens` | `String` | Lens identifier |
| `payloadIndex` | `String` | Payload index |
| `zoom` | `Float` | Target zoom factor |

---

## Live Streaming Models

### LiveStreamStartRequest

Request to start a video live stream.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `videoId` | `String` | Video stream identifier |
| `streamServer` | `String` | Target streaming server URL |
| `videoType` | `String` | Stream type (e.g., RTMP, RTSP) |

---

### LiveStreamStopRequest

Request to stop a video live stream.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Device serial number |
| `tid` | `String` | Transaction identifier |
| `videoId` | `String` | Video stream identifier to stop |

---

## Telemetry Models

### TelemetryRequestData

Top-level wrapper for pushing telemetry data to the Live Data Service.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `tid` | `String` | Transaction identifier |
| `sn` | `String` | Device serial number |
| `assetId` | `String` | Platform asset identifier |
| `timestamp` | `LocalDateTime` | When the telemetry was recorded |
| `type` | `LiveDataType` | `ASSET_TELEMETRY` or `SUBASSET_TELEMETRY` |
| `assetTelemetry` | `AssetTelemetryData` | Asset-level telemetry data |
| `subAssetTelemetry` | `SubAssetTelemetryData` | Sub-asset-level telemetry data |

Supports the `@Builder` pattern:

```java
TelemetryRequestData data = TelemetryRequestData.builder()
    .sn("DEVICE_SN")
    .tid(UUID.randomUUID().toString())
    .timestamp(LocalDateTime.now())
    .type(LiveDataType.ASSET_TELEMETRY)
    .assetTelemetry(assetData)
    .build();
```

---

### AssetTelemetryData

Telemetry for the primary asset (dock, station, or ground device).

**Package:** `com.zqnt.utils.edge.sdk.domains`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Telemetry identifier |
| `timestamp` | `LocalDateTime` | Measurement time |
| `latitude` | `Float` | GPS latitude |
| `longitude` | `Float` | GPS longitude |
| `absoluteAltitude` | `Float` | Altitude above sea level (m) |
| `relativeAltitude` | `Float` | Altitude above ground (m) |
| `environmentTemp` | `Float` | Ambient temperature (C) |
| `insideTemp` | `Float` | Internal temperature (C) |
| `humidity` | `Float` | Humidity (%) |
| `mode` | `AssetModes` | Operational mode |
| `rainfall` | `RainfallEnum` | Rainfall condition |
| `heading` | `Float` | Heading (degrees) |
| `debugModeOpen` | `Boolean` | Debug mode active |
| `hasActiveManualControlSession` | `Boolean` | Manual control active |
| `coverState` | `AssetCoverStateEnum` | Cover state |
| `workingVoltage` | `Integer` | Working voltage (mV) |
| `workingCurrent` | `Integer` | Working current (mA) |
| `supplyVoltage` | `Integer` | Supply voltage (mV) |
| `windSpeed` | `Float` | Wind speed (m/s) |
| `positionValid` | `Boolean` | GPS position valid |
| `manualControlState` | `ManualControlStateEnum` | DRC state |
| `subAssetInformation` | `SubAssetInformation` | Paired sub-asset info |
| `subAssetAtHome` | `Boolean` | Sub-asset at dock |
| `subAssetCharging` | `Boolean` | Sub-asset charging |
| `subAssetPercentage` | `Float` | Sub-asset battery (%) |
| `networkInformation` | `NetworkInformation` | Network info |
| `airConditioner` | `AirConditioner` | AC state |

**Nested types:**

`SubAssetInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `sn` | `String` | Sub-asset serial number |
| `model` | `String` | Sub-asset model name |
| `paired` | `Boolean` | Whether paired |
| `online` | `Boolean` | Whether online |

`NetworkInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `type` | `NetworkTypeEnum` | Network type (4G, 5G, etc.) |
| `rate` | `Float` | Data rate |
| `quality` | `NetworkStateQualityEnum` | Signal quality |

`AirConditioner`:

| Field | Type | Description |
|-------|------|-------------|
| `state` | `AssetAirConditionerStateEnum` | AC state |
| `switchTime` | `Integer` | Time until state switch |

---

### SubAssetTelemetryData

Telemetry for a sub-asset (drone, vehicle).

**Package:** `com.zqnt.utils.edge.sdk.domains`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Telemetry identifier |
| `timestamp` | `LocalDateTime` | Measurement time |
| `latitude` | `Float` | GPS latitude |
| `longitude` | `Float` | GPS longitude |
| `absoluteAltitude` | `Float` | Altitude above sea level (m) |
| `relativeAltitude` | `Float` | Altitude above ground (m) |
| `horizontalSpeed` | `Float` | Horizontal speed (m/s) |
| `verticalSpeed` | `Float` | Vertical speed (m/s) |
| `windSpeed` | `Float` | Wind speed (m/s) |
| `windDirection` | `String` | Wind direction |
| `heading` | `Float` | Heading (degrees) |
| `gear` | `Integer` | Landing gear state |
| `heightLimit` | `Integer` | Max height limit (m) |
| `homeDistance` | `Float` | Distance to home (m) |
| `totalMovementDistance` | `Double` | Total distance traveled (m) |
| `totalMovementTime` | `Double` | Total flight time (s) |
| `mode` | `SubAssetMode` | Flight mode |
| `country` | `String` | Country code |
| `batteryInformation` | `BatteryInformation` | Battery data |
| `payloadTelemetry` | `PayloadTelemetry` | Payload/camera data |

**Nested types:**

`BatteryInformation`:

| Field | Type | Description |
|-------|------|-------------|
| `percentage` | `String` | Battery percentage |
| `remainingTime` | `Integer` | Remaining time (seconds) |
| `returnToHomePower` | `String` | Power needed for RTH |

`PayloadTelemetry`:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Payload identifier |
| `timestamp` | `LocalDateTime` | Measurement time |
| `name` | `String` | Payload name |
| `cameraData` | `CameraData` | Camera state |
| `rangeFinderData` | `RangeFinderData` | Rangefinder data |
| `sensorData` | `SensorData` | Sensor readings |

`CameraData`:

| Field | Type | Description |
|-------|------|-------------|
| `currentLens` | `String` | Active lens |
| `gimbalPitch` | `Float` | Gimbal pitch (degrees) |
| `gimbalYaw` | `Float` | Gimbal yaw (degrees) |
| `zoomFactor` | `Float` | Current zoom level |

`RangeFinderData`:

| Field | Type | Description |
|-------|------|-------------|
| `targetLatitude` | `Float` | Target latitude |
| `targetLongitude` | `Float` | Target longitude |
| `targetDistance` | `Float` | Distance to target (m) |
| `targetAltitude` | `Float` | Target altitude (m) |

`SensorData`:

| Field | Type | Description |
|-------|------|-------------|
| `targetTemperature` | `Float` | Measured temperature (C) |

---

## Mission Models

### MissionData

Represents a mission returned by the Mission Autonomy Service.

**Package:** `com.zqnt.sdk.edge.adapter.domains`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `UUID` | Unique mission identifier |
| `name` | `String` | Mission name |

---

## Configuration Models

### EdgeClientConfig

The edge adapter configuration interface, mapped from `application.properties` with `@ConfigMapping(prefix = "zequent.edge")`.

**Package:** `com.zqnt.sdk.edge.config`

| Method | Return Type | Property | Default | Description |
|--------|-------------|----------|---------|-------------|
| `endpoint()` | `String` | `zequent.edge.endpoint` | -- | Adapter listen address |
| `sn()` | `String` | `zequent.edge.sn` | -- | Device serial number |
| `timeout()` | `Duration` | `zequent.edge.timeout` | `30s` | Command timeout |
| `maxRetries()` | `int` | `zequent.edge.max-retries` | `3` | Max retry attempts |
| `assetType()` | `AssetTypeEnum` | `zequent.edge.asset-type` | -- | Asset type |
| `assetVendor()` | `AssetVendor` | `zequent.edge.asset-vendor` | -- | Asset vendor |
| `assetId()` | `Optional<String>` | `zequent.edge.asset-id` | empty | Platform asset ID |

Access these values in your code via the `EdgeClient` bean:

```java
@Inject
EdgeClient edgeClient;

String sn = edgeClient.getSn();
EdgeClientConfig config = edgeClient.getConfig();
Duration timeout = config.timeout();
```

Or inject `EdgeClientConfig` directly:

```java
@Inject
EdgeClientConfig config;

String sn = config.sn();
AssetTypeEnum type = config.assetType();
```
