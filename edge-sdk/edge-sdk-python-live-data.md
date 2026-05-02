# Edge SDK (Python) — Live Data

The Live Data side of the Python Edge SDK is `TelemetryPublisher`. It owns a long-lived gRPC streaming connection to the Live Data service and lets you push asset and sub-asset telemetry as plain dataclasses.

For Java, see [edge-sdk-live-data.md](edge-sdk-live-data.md).

---

## Lifecycle

```python
from edge_sdk import TelemetryPublisher

pub = TelemetryPublisher(host="localhost", port=8003, sn="DOCK-1")
await pub.connect()
try:
    await pub.publish_asset_telemetry(asset_telemetry)
    await pub.publish_sub_asset_telemetry(sub_asset_telemetry)
finally:
    await pub.close()
```

`connect()` is idempotent. The publisher reconnects automatically if the underlying stream fails — your code typically just loops and pushes telemetry at whatever cadence makes sense for your device.

`from_env()` reads `LIVE_DATA_SERVICE_HOST` / `LIVE_DATA_SERVICE_PORT`:

```python
pub = TelemetryPublisher.from_env(sn="DOCK-1")
```

---

## Asset telemetry

`AssetTelemetry` describes the top-level asset (the dock, the drone itself when used standalone, etc.).

```python
from datetime import datetime, timezone
from edge_sdk import (
    AssetTelemetry, AssetPositionState, AssetNetworkInfo,
    NetworkType, NetworkStateQuality, AssetMode,
)

asset = AssetTelemetry(
    id="DOCK-1",
    timestamp=datetime.now(tz=timezone.utc),
    position=AssetPositionState(
        latitude=47.3769, longitude=8.5417, altitude=450.0,
        heading=180.0,
    ),
    mode=AssetMode.WORKING,
    battery_percentage=87,
    environment_temperature=22.5,
    humidity=65.0,
    network=AssetNetworkInfo(
        type=NetworkType.LTE,
        quality=NetworkStateQuality.STRONG,
        rssi=-72,
    ),
)
await pub.publish_asset_telemetry(asset)
```

All fields are optional — populate only what your device measures. The publisher omits `None` fields from the wire.

---

## Sub-asset telemetry

`SubAssetTelemetry` describes payloads / drones-inside-a-dock / cameras / sensors:

```python
from edge_sdk import (
    SubAssetTelemetry, SubAssetMode, SubAssetBatteryInfo,
    PayloadTelemetry, CameraData,
)

sub = SubAssetTelemetry(
    id="DRONE-1",
    parent_id="DOCK-1",
    timestamp=datetime.now(tz=timezone.utc),
    mode=SubAssetMode.FLYING,
    battery=SubAssetBatteryInfo(percentage=72, voltage_mv=23_400, temperature=31.0),
    payload=PayloadTelemetry(
        camera=CameraData(zoom_factor=2.5, lens="WIDE"),
    ),
)
await pub.publish_sub_asset_telemetry(sub)
```

---

## Detection results

For tasks like `DetectTask`, attach detection output to a `DetectionResponse`:

```python
from edge_sdk import DetectionResponse, DetectionResult, BoundingBox

await pub.publish_detection(
    DetectionResponse(
        task_id="task-uuid",
        timestamp=datetime.now(tz=timezone.utc),
        results=[
            DetectionResult(
                class_name="person",
                confidence=0.91,
                box=BoundingBox(x=120, y=80, width=64, height=128),
            ),
        ],
    )
)
```

---

## Error handling and reconnection

`publish_*` methods raise `grpc.aio.AioRpcError` if the call fails after the publisher has exhausted its internal retry budget. Treat them as transient and back off your producer loop:

```python
import asyncio
import grpc

while True:
    try:
        await pub.publish_asset_telemetry(read_from_device())
    except grpc.aio.AioRpcError as e:
        log.warning("telemetry push failed: %s; backing off", e.code())
        await asyncio.sleep(1.0)
    await asyncio.sleep(0.1)
```

The publisher itself reconnects when the server drops the stream — you don't need to call `connect()` again.

---

## Performance tips

- **Push at a fixed cadence**, not on every sensor reading. 1–10 Hz is typical for asset telemetry; 0.5–2 Hz for sub-asset.
- **Reuse a single publisher** for an entire process. Don't create one per push.
- **Keep dataclass construction off the hot path** if you read sensors at a higher rate than you publish.
- **Use `asyncio.gather`** if you need to push asset and sub-asset telemetry concurrently from independent tasks.

---

## Sending telemetry from inside an adapter method

You can hold a reference to the publisher on your `EdgeAdapter` subclass:

```python
class MyAdapter(EdgeAdapter):
    def __init__(self, telemetry: TelemetryPublisher):
        self._telemetry = telemetry

    async def take_off(self, ctx, coordinates):
        result = await drone.takeoff(...)
        await self._telemetry.publish_asset_telemetry(result.to_asset_telemetry())
        return EdgeResponse.success(ctx.tid, ctx.sn, "Takeoff initiated")
```

Wire it in your `main`:

```python
pub = TelemetryPublisher.from_env(sn=...)
await pub.connect()
adapter = MyAdapter(telemetry=pub)
server = EdgeServer(adapter=adapter, port=9001)
await server.serve()
```
