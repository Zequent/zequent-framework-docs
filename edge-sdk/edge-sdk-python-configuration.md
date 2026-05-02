# Edge SDK (Python) — Configuration

The Python Edge SDK is configured exclusively via **environment variables**. There is no `application.properties` equivalent — the SDK reads `os.environ` lazily in factory helpers like `EdgeServer.from_env()`, `TelemetryPublisher.from_env()`, and `ConnectorClient.from_env()`.

For Java/Quarkus configuration see [edge-sdk-configuration.md](edge-sdk-configuration.md).

---

## Edge identity

| Variable                     | Description                                                                                  | Default     |
|------------------------------|----------------------------------------------------------------------------------------------|-------------|
| `ZEQUENT_EDGE_ENDPOINT`      | Public `host:port` the platform should reach this adapter on. Used during registration.      | _required_  |
| `ZEQUENT_EDGE_SN`            | Serial number / unique identifier of the asset.                                              | _required_  |
| `ZEQUENT_EDGE_ASSET_TYPE`    | One of `ASSET_TYPE_DOCK`, `ASSET_TYPE_DRONE`, `ASSET_TYPE_VEHICLE`, etc.                     | _required_  |
| `ZEQUENT_EDGE_ASSET_VENDOR`  | Vendor enum: `DJI`, `AUTEL`, `PARROT`, `CUSTOM`, ...                                         | `CUSTOM`    |
| `ZEQUENT_EDGE_ASSET_ID`      | Optional pre-existing asset UUID. Set this when re-using an asset registered out-of-band.    | _unset_     |

---

## gRPC server

| Variable                       | Description                                                          | Default   |
|--------------------------------|----------------------------------------------------------------------|-----------|
| `ZEQUENT_EDGE_HOST`            | Bind address for the local gRPC server.                              | `0.0.0.0` |
| `ZEQUENT_EDGE_PORT`            | Bind port for the local gRPC server.                                 | `9001`    |
| `ZEQUENT_EDGE_MAX_WORKERS`     | Worker thread pool size (only relevant for sync code paths).         | `10`      |
| `ZEQUENT_EDGE_HEALTH_ENABLED`  | Set to `false` to disable the gRPC health-check service.             | `true`    |

You can also build an `EdgeServer` programmatically:

```python
EdgeServer(adapter=MyAdapter(), host="0.0.0.0", port=9001)
```

---

## Live Data service

Used by `TelemetryPublisher`.

| Variable                       | Description                                  | Default     |
|--------------------------------|----------------------------------------------|-------------|
| `LIVE_DATA_SERVICE_HOST`       | Live Data service hostname.                  | `localhost` |
| `LIVE_DATA_SERVICE_PORT`       | Live Data service gRPC port.                 | `8003`      |
| `LIVE_DATA_KEEP_ALIVE_SEC`     | gRPC keepalive interval in seconds.          | `30`        |

```python
pub = TelemetryPublisher.from_env(sn="DOCK-1")
```

---

## Connector service

Used by `ConnectorClient` to register assets / look up resources.

| Variable                     | Description                              | Default     |
|------------------------------|------------------------------------------|-------------|
| `CONNECTOR_SERVICE_HOST`     | Connector service hostname.              | `localhost` |
| `CONNECTOR_SERVICE_PORT`     | Connector service gRPC port.             | `8010`      |

```python
async with ConnectorClient.from_env() as conn:
    await conn.register_asset(...)
```

---

## Optional Redis registration

If `REDIS_URL` is set, the SDK can publish a self-registration record to Redis at startup so the platform discovers it dynamically.

| Variable                      | Description                                                  | Default                |
|-------------------------------|--------------------------------------------------------------|------------------------|
| `REDIS_URL`                   | Redis connection URL.                                        | _unset (registration off)_ |
| `ZEQUENT_REGISTRATION_TTL_SEC`| TTL of the registration record.                              | `60`                   |
| `ZEQUENT_REGISTRATION_REFRESH_SEC` | Period at which the record is refreshed.                | `30`                   |

To enable, pass a `RegistrationConfig` to `EdgeServer`:

```python
from edge_sdk import EdgeServer, RegistrationConfig

server = EdgeServer(
    adapter=MyAdapter(),
    port=9001,
    registration=RegistrationConfig.from_env(),
)
```

---

## Logging

The SDK uses the standard `logging` module under the namespace `edge_sdk.*`. No env variables are introduced; configure with `logging.basicConfig(...)` or `logging.config.dictConfig(...)` as usual.

```python
import logging
logging.basicConfig(level=logging.INFO)
logging.getLogger("edge_sdk").setLevel(logging.DEBUG)
```

---

## TLS / authentication

The SDK uses insecure gRPC channels by default for parity with the local-development workflow. To enable TLS or auth, supply a custom `grpc.aio.Channel` to `TelemetryPublisher` / `ConnectorClient`:

```python
import grpc
creds = grpc.ssl_channel_credentials()
channel = grpc.aio.secure_channel("livedata.example.com:443", creds)

pub = TelemetryPublisher(channel=channel, sn="DOCK-1")
```

For per-call metadata (bearer tokens, etc.), pass `metadata=[(...)]` to `connect()` — the publisher forwards it on every stream.

---

## Putting it together

```python
import asyncio
from edge_sdk import EdgeServer, RegistrationConfig, TelemetryPublisher

from .adapter import MyDeviceAdapter

async def main():
    adapter = MyDeviceAdapter()
    pub = TelemetryPublisher.from_env(sn=os.environ["ZEQUENT_EDGE_SN"])
    await pub.connect()

    server = EdgeServer.from_env(
        adapter=adapter,
        registration=RegistrationConfig.from_env(),
    )
    try:
        await server.serve()
    finally:
        await pub.close()

asyncio.run(main())
```
