# Edge SDK -- Live Data Service

The `LiveDataService` interface manages persistent gRPC telemetry streams between the edge adapter and the platform's Live Data Service. It provides both a POJO-based API (recommended for most use cases) and a raw Proto-based API for advanced scenarios.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [POJO-based API (Recommended)](#pojo-based-api-recommended)
- [Proto-based API (Advanced)](#proto-based-api-advanced)
- [Stream Management](#stream-management)
- [Telemetry Data Models](#telemetry-data-models)
  - [TelemetryRequestData](#telemetryrequestdata)
  - [AssetTelemetryData](#assettelemetrydata)
  - [SubAssetTelemetryData](#subassettelemetrydata)
- [Configuration](#configuration)
- [Best Practices](#best-practices)

---

## Overview

Live telemetry data is the real-time heartbeat of every asset in the Zequent ecosystem. Edge adapters continuously push telemetry -- position, battery, environmental readings, camera state, and more -- to the Live Data Service, where it is broadcast to Client SDK consumers and stored for historical analysis.

The `LiveDataService` abstracts the complexity of managing gRPC streams. Internally, it creates one `BroadcastProcessor` per device serial number and subscribes it to the Live Data Service gRPC stub. Streams reconnect automatically on failure with exponential backoff.

---

## How It Works

```
Your Adapter Code
      |
      v
LiveDataService.produceTelemetryData(TelemetryRequestData)
      |
      v
TelemetryMapper (POJO --> Proto)
      |
      v
BroadcastProcessor (one per device)
      |
      v
gRPC stream --> Live Data Service (platform)
```

Key characteristics of the implementation:

- **One stream per device.** A `BroadcastProcessor` is created for each unique serial number and reused for all subsequent telemetry pushes.
- **Automatic reconnection.** If the gRPC stream fails, it retries with exponential backoff (1s to 30s, 20% jitter, up to 10 attempts). If all retries are exhausted, it schedules a reconnect after 30 seconds.
- **Graceful shutdown.** All streams are closed on application shutdown via the `@PreDestroy` lifecycle callback.
- **Thread-safe.** Device-to-processor mappings are stored in a `ConcurrentHashMap`, and shutdown state is tracked with an `AtomicBoolean`.

---

## POJO-based API (Recommended)

### 1. Build the Telemetry Data Object

Map data from your device into the `TelemetryRequestData` POJO:

```java
import com.zequent.framework.edge.sdk.models.TelemetryRequestData;
import com.zequent.framework.common.proto.LiveDataType;
import com.zequent.framework.utils.edge.sdk.dto.AssetTelemetryData;
import java.time.LocalDateTime;

TelemetryRequestData data = TelemetryRequestData.builder()
    .sn("YOUR_DEVICE_SN")
    .tid(UUID.randomUUID().toString())
    .assetId("asset-uuid")
    .timestamp(LocalDateTime.now())
    .type(LiveDataType.ASSET_TELEMETRY)
    .assetTelemetry(assetTelemetry)  // see models section below
    .build();
```

### 2. Send It

Reactive (Mutiny):

```java
import com.zequent.framework.edge.sdk.interfaces.LiveDataService;

private final LiveDataService liveDataService;

public void sendTelemetry(TelemetryRequestData data) {
    liveDataService.produceTelemetryData(data)
        .subscribe().with(
            ignored -> log.debug("Telemetry sent for {}", data.getSn()),
            err -> log.error("Error sending telemetry", err)
        );
}
```

Future-based (CompletionStage):

```java
public void sendTelemetry(TelemetryRequestData data) {
    liveDataService.produceTelemetryData(data)
        .subscribeAsCompletionStage()
        .thenRun(() -> log.debug("Telemetry sent"))
        .exceptionally(err -> {
            log.error("Error sending telemetry", err);
            return null;
        });
}
```

---

## Proto-based API (Advanced)

For scenarios where you need full control over the Proto message (e.g., you already have telemetry data in Proto format from another system), use the direct Proto API:

```java
import com.zequent.framework.services.livedata.proto.ProduceTelemetryRequest;

ProduceTelemetryRequest protoRequest = ProduceTelemetryRequest.newBuilder()
    .setBase(RequestBase.newBuilder()
        .setSn("YOUR_DEVICE_SN")
        .setTid(UUID.randomUUID().toString())
        .setTimestamp(ProtobufHelpers.now())
        .build())
    // ... set telemetry fields
    .build();

liveDataService.produceTelemetry("YOUR_DEVICE_SN", protoRequest)
    .subscribe().with(
        ignored -> log.debug("Proto telemetry sent"),
        err -> log.error("Error", err)
    );
```

Both APIs share the same underlying stream infrastructure, so there is no performance difference.

---

## Stream Management

### Close a Single Device Stream

```java
liveDataService.closeStream("YOUR_DEVICE_SN")
    .subscribe().with(
        ignored -> log.info("Stream closed for device"),
        err -> log.error("Error closing stream", err)
    );
```

### Close All Streams

Typically called during shutdown, but can be triggered manually:

```java
liveDataService.closeAllStreams()
    .subscribe().with(
        ignored -> log.info("All streams closed"),
        err -> log.error("Error closing streams", err)
    );
```

The `LiveDataServiceImpl` also registers a `@PreDestroy` callback that automatically closes all streams with a 10-second timeout on application shutdown.

---

## Telemetry Data Models

### TelemetryRequestData

The top-level wrapper that carries one telemetry payload:

| Field | Type | Description |
|-------|------|-------------|
| `tid` | `String` | Transaction identifier for tracing |
| `sn` | `String` | Device serial number |
| `assetId` | `String` | Platform asset identifier |
| `timestamp` | `LocalDateTime` | When the telemetry was recorded |
| `type` | `LiveDataType` | `ASSET_TELEMETRY` or `SUBASSET_TELEMETRY` |
| `assetTelemetry` | `AssetTelemetryData` | Asset-level telemetry (set when type is `ASSET_TELEMETRY`) |
| `subAssetTelemetry` | `SubAssetTelemetryData` | Sub-asset-level telemetry (set when type is `SUBASSET_TELEMETRY`) |

### AssetTelemetryData

Telemetry data for the primary asset (e.g., a dock or ground station):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Asset telemetry identifier |
| `timestamp` | `LocalDateTime` | Measurement timestamp |
| `latitude` | `Float` | GPS latitude |
| `longitude` | `Float` | GPS longitude |
| `absoluteAltitude` | `Float` | Altitude above sea level (meters) |
| `relativeAltitude` | `Float` | Altitude above takeoff point (meters) |
| `environmentTemp` | `Float` | Ambient temperature (Celsius) |
| `insideTemp` | `Float` | Internal temperature (Celsius) |
| `humidity` | `Float` | Humidity percentage |
| `mode` | `AssetModes` | Current operational mode |
| `rainfall` | `RainfallEnum` | Rainfall condition |
| `heading` | `Float` | Heading in degrees |
| `debugModeOpen` | `Boolean` | Whether debug mode is active |
| `hasActiveManualControlSession` | `Boolean` | Whether manual control is active |
| `coverState` | `AssetCoverStateEnum` | Dock cover state (open/closed/moving) |
| `workingVoltage` | `Integer` | System voltage (mV) |
| `workingCurrent` | `Integer` | System current (mA) |
| `supplyVoltage` | `Integer` | Supply voltage (mV) |
| `windSpeed` | `Float` | Wind speed (m/s) |
| `positionValid` | `Boolean` | Whether GPS position is valid |
| `manualControlState` | `ManualControlStateEnum` | DRC manual control state |
| `subAssetInformation` | `SubAssetInformation` | Paired sub-asset info (sn, model, paired, online) |
| `subAssetAtHome` | `Boolean` | Whether the sub-asset is at the dock |
| `subAssetCharging` | `Boolean` | Whether the sub-asset is charging |
| `subAssetPercentage` | `Float` | Sub-asset battery percentage |
| `networkInformation` | `NetworkInformation` | Network type, rate, quality |
| `airConditioner` | `AirConditioner` | Air conditioner state and switch time |

### SubAssetTelemetryData

Telemetry data for a sub-asset (e.g., a drone):

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Sub-asset telemetry identifier |
| `timestamp` | `LocalDateTime` | Measurement timestamp |
| `latitude` | `Float` | GPS latitude |
| `longitude` | `Float` | GPS longitude |
| `absoluteAltitude` | `Float` | Altitude above sea level (meters) |
| `relativeAltitude` | `Float` | Altitude above takeoff point (meters) |
| `horizontalSpeed` | `Float` | Horizontal speed (m/s) |
| `verticalSpeed` | `Float` | Vertical speed (m/s) |
| `windSpeed` | `Float` | Wind speed (m/s) |
| `windDirection` | `String` | Wind direction |
| `heading` | `Float` | Heading in degrees |
| `gear` | `Integer` | Landing gear state |
| `heightLimit` | `Integer` | Maximum height limit (meters) |
| `homeDistance` | `Float` | Distance to home point (meters) |
| `totalMovementDistance` | `Double` | Total distance traveled (meters) |
| `totalMovementTime` | `Double` | Total flight time (seconds) |
| `mode` | `SubAssetMode` | Current flight mode |
| `country` | `String` | Country code |
| `batteryInformation` | `BatteryInformation` | Battery percentage, remaining time, RTH power |
| `payloadTelemetry` | `PayloadTelemetry` | Camera, rangefinder, and sensor data |

#### PayloadTelemetry

| Field | Type | Description |
|-------|------|-------------|
| `id` | `String` | Payload identifier |
| `timestamp` | `LocalDateTime` | Measurement timestamp |
| `name` | `String` | Payload name |
| `cameraData` | `CameraData` | Current lens, gimbal pitch/yaw, zoom factor |
| `rangeFinderData` | `RangeFinderData` | Target lat/lon/distance/altitude |
| `sensorData` | `SensorData` | Target temperature |

---

## Configuration

The Live Data Service gRPC client is configured in `application.properties`:

```properties
quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003
quarkus.grpc.clients.live-data-service.keep-alive-without-calls=true
```

For container deployments:

```properties
quarkus.grpc.clients.live-data-service.host=live-data-service
quarkus.grpc.clients.live-data-service.port=8003
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.

---

## Best Practices

1. **Send telemetry at a reasonable frequency.** Sending too fast can overwhelm the gRPC stream and the platform. For most assets, 1-5 Hz is appropriate. For OSD (on-screen display) data from drones, the typical rate is 2 Hz.

2. **Use the POJO API unless you have a specific reason not to.** The `TelemetryMapper` handles all Proto conversions for you, including timestamp mapping between `LocalDateTime` and Protobuf `Timestamp`.

3. **Set the correct `LiveDataType`.** Use `ASSET_TELEMETRY` for dock/station data and `SUBASSET_TELEMETRY` for drone/vehicle data. This determines how the data is routed and stored on the platform.

4. **Include a transaction ID.** Setting `tid` on every telemetry message enables end-to-end tracing across the system.

5. **Do not manually manage streams.** Let the SDK handle stream creation, reconnection, and teardown. If you need to reset a stream, call `closeStream(deviceSn)` and the next `produceTelemetry` call will create a new one automatically.

6. **Handle the shutdown gracefully.** If your adapter has its own shutdown logic, call `closeAllStreams()` before tearing down other resources. The SDK does this automatically via `@PreDestroy`, but explicit ordering can prevent race conditions.
