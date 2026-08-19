# Edge SDK (Python) — Configuration

The Python Edge SDK is configured exclusively via **environment variables**, centralized in `EdgeAdapterConfig`. Call `EdgeAdapterConfig.from_env()` to build one, then `.runtime()` to get a connected `EdgeAdapterRuntime` — see the [Quickstart](edge-sdk-python-quickstart.md).

For Java/Quarkus configuration see [edge-sdk-configuration.md](edge-sdk-configuration.md).

---

## `EdgeAdapterConfig` environment variables

| Variable | Description | Default |
|----------|--------------|---------|
| `GRPC_HOST` | Bind address for this adapter's own gRPC server | `0.0.0.0` |
| `GRPC_PORT` | Bind port for this adapter's own gRPC server | `50051` |
| `CONNECTOR_HOST` | Connector service hostname | `localhost` |
| `CONNECTOR_PORT` | Connector service gRPC port | `50053` |
| `TELEMETRY_HOST` | Live Data service hostname | `localhost` |
| `TELEMETRY_PORT` | Live Data service gRPC port | `50052` |
| `MISSION_AUTONOMY_HOST` | Mission Autonomy service hostname | `localhost` |
| `MISSION_AUTONOMY_PORT` | Mission Autonomy service gRPC port | `50054` |
| `ADAPTER_SN` | Serial number this adapter logs under; each telemetry frame also carries its own asset SN, so this doesn't need to match for multi-asset adapters | `""` |
| `LOG_LEVEL` | Python log level name | `INFO` |
| `LOG_FORMAT` | `json` or `text` | `json` |

**The library's built-in port defaults do not match the platform's real service ports.** Always set `CONNECTOR_PORT=8010`, `TELEMETRY_PORT=8003`, and `MISSION_AUTONOMY_PORT=8004` (or your deployment's actual ports) explicitly — don't rely on the defaults above.

```python
from edge_sdk import EdgeAdapterConfig

config = EdgeAdapterConfig.from_env()
async with config.runtime() as runtime:
    ...
```

For tests or explicit configuration, construct `EdgeAdapterConfig(...)` directly instead of calling `from_env()` — every field has a keyword-argument equivalent.

---

## Optional: automatic Redis service-discovery registration

If set, the SDK registers this adapter's endpoint in Redis on startup (`online=True`) and marks it offline on shutdown, so the platform's client-side load balancer can discover it dynamically.

| Variable | Description | Default |
|----------|--------------|---------|
| `EDGE_ENDPOINT` | gRPC endpoint advertised to the platform, e.g. `grpc://my-adapter.internal:9001` | unset (registration off) |
| `ASSET_TYPE` | Proto-style asset type name, e.g. `ASSET_TYPE_AIRCRAFT` | unset |
| `ASSET_VENDOR` | Proto-style vendor name, e.g. `ASSET_VENDOR_MAVLINK` | unset |
| `REDIS_URL` | Redis connection URL | `redis://localhost:6379` |

Registration only activates when `EDGE_ENDPOINT`, `ASSET_TYPE`, and `ASSET_VENDOR` are all set — `EdgeAdapterConfig.runtime()`'s `serve()` builds the `RegistrationConfig` for you automatically in that case. To configure it directly:

```python
from edge_sdk import RegistrationConfig

registration = RegistrationConfig.from_env()  # reads EDGE_ENDPOINT / ASSET_TYPE / ASSET_VENDOR / REDIS_URL
```

Redis key format: `edge-endpoints:{VENDOR}` (matches the Java SDK's equivalent cache key).

---

## Logging

`EdgeAdapterRuntime` calls `logging.basicConfig(...)` for you based on `LOG_LEVEL`/`LOG_FORMAT` — you don't need to configure it yourself unless you want something different. `LOG_FORMAT=json` uses `python-json-logger` if installed, falling back to plain text otherwise.

```python
import logging
logging.getLogger("edge_sdk").setLevel(logging.DEBUG)
```

---

## TLS / authentication

The SDK uses insecure gRPC channels by default for parity with the local-development workflow. To enable TLS or auth, construct the individual clients (`ConnectorClient`, `TelemetryPublisher`, `MissionAutonomyClient`) with a custom `grpc.aio.Channel` instead of going through `EdgeAdapterConfig.runtime()`:

```python
import grpc
creds = grpc.ssl_channel_credentials()
channel = grpc.aio.secure_channel("livedata.example.com:443", creds)

pub = TelemetryPublisher(channel=channel, sn="DOCK-1")
```

For per-call metadata (bearer tokens, etc.), pass `metadata=[(...)]` to `connect()` where supported.

---

## Putting it together

```python
import asyncio
from edge_sdk import EdgeAdapterConfig

from .adapter import MyDeviceAdapter

async def main():
    config = EdgeAdapterConfig.from_env()
    async with config.runtime() as runtime:
        adapter = MyDeviceAdapter()
        await runtime.serve(adapter)

asyncio.run(main())
```
