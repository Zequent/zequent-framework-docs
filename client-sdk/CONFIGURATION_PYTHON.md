# Zequent Client SDK (Python) - Configuration

The Python Client SDK is configured exclusively via **environment variables** read by `ZequentClient.from_env()` and `ZequentClientConfig.from_env()`. There is no `application.properties` equivalent and no DI container.

For Java/Quarkus configuration see [CONFIGURATION.md](CONFIGURATION.md).

---

## Service endpoints

| Variable                          | Default     | Description                                  |
|-----------------------------------|-------------|----------------------------------------------|
| `REMOTE_CONTROL_SERVICE_HOST`     | `localhost` | Hostname of the Remote Control service       |
| `REMOTE_CONTROL_SERVICE_PORT`     | `8002`      | gRPC port                                    |
| `MISSION_AUTONOMY_SERVICE_HOST`   | `localhost` | Hostname of the Mission Autonomy service     |
| `MISSION_AUTONOMY_SERVICE_PORT`   | `8004`      | gRPC port                                    |
| `LIVE_DATA_SERVICE_HOST`          | `localhost` | Hostname of the Live Data service            |
| `LIVE_DATA_SERVICE_PORT`          | `8003`      | gRPC port                                    |

```python
from client_sdk import ZequentClient

async with ZequentClient.from_env() as client:
    ...
```

---

## Programmatic configuration

When env vars aren't a fit (multi-tenant apps, dynamic endpoints, tests):

```python
from client_sdk import ZequentClient
from client_sdk.config import ZequentClientConfig

config = ZequentClientConfig(
    remote_control_host="rc.example.com", remote_control_port=8002,
    mission_autonomy_host="ma.example.com", mission_autonomy_port=8004,
    live_data_host="ld.example.com", live_data_port=8003,
)
async with ZequentClient(config) as client:
    ...
```

`ZequentClientConfig.from_env()` is also available if you want to take the env defaults and override a few fields:

```python
config = ZequentClientConfig.from_env()
config.live_data_host = "live.example.com"
```

---

## Resilience configuration

Unary RPCs are wrapped with retry + circuit-breaker policies. Defaults are sane; override via `ResilienceConfig`:

```python
from client_sdk.config.resilience import ResilienceConfig

config.resilience = ResilienceConfig(
    max_attempts=5,                  # total attempts including the initial call
    initial_backoff_ms=200,          # first retry delay
    max_backoff_ms=5_000,            # cap on exponential backoff
    backoff_multiplier=2.0,
    breaker_failure_threshold=10,    # consecutive failures before tripping
    breaker_reset_seconds=30,        # half-open window
)
```

Retryable gRPC status codes:

- `UNAVAILABLE`
- `DEADLINE_EXCEEDED`
- `RESOURCE_EXHAUSTED`
- `ABORTED`

Non-retryable codes (propagated immediately):

- `INVALID_ARGUMENT`
- `FAILED_PRECONDITION`
- `NOT_FOUND`
- `ALREADY_EXISTS`
- `PERMISSION_DENIED`
- `UNAUTHENTICATED`

Streaming RPCs do **not** retry transparently — the application must re-subscribe.

---

## Per-call deadlines

`grpc.aio` deadlines are honoured for every unary call. Set a per-call deadline by passing `timeout=` to the SDK method:

```python
await client.remote_control.takeoff(req, timeout=5.0)
```

`timeout` is in seconds. If unset, the SDK uses the default channel deadline (none).

---

## Logging

Standard `logging` namespaces:

- `client_sdk.zequent_client` — top-level lifecycle
- `client_sdk.grpc_.resilience` — retry + breaker decisions
- `client_sdk.{remote_control,mission_autonomy,live_data}.client` — per-RPC logs

```python
import logging
logging.basicConfig(level=logging.INFO)
logging.getLogger("client_sdk.grpc_.resilience").setLevel(logging.DEBUG)
```

The SDK never logs sensitive request bodies; only operation name + outcome + status code.

---

## TLS, auth, and custom channels

By default the SDK uses **insecure** gRPC channels for parity with local development. To use TLS or auth, pass a pre-built `grpc.aio.Channel` per service:

```python
import grpc
from client_sdk import ZequentClient
from client_sdk.config import ZequentClientConfig

creds = grpc.ssl_channel_credentials()

config = ZequentClientConfig(
    remote_control_channel=grpc.aio.secure_channel("rc.example.com:443", creds),
    mission_autonomy_channel=grpc.aio.secure_channel("ma.example.com:443", creds),
    live_data_channel=grpc.aio.secure_channel("ld.example.com:443", creds),
)
async with ZequentClient(config) as client:
    ...
```

For per-call metadata (e.g. JWT bearers), pass `metadata=[(...)]` to SDK methods or attach a gRPC interceptor to the channel.

---

## Connection pooling

Each sub-client owns one `grpc.aio` channel. `grpc.aio` channels multiplex unbounded concurrent calls over HTTP/2, so a single `ZequentClient` instance scales to thousands of concurrent calls without further pooling.

**Do not create one `ZequentClient` per request** — keep it as an application singleton (per the lifespan pattern in the [Quickstart](QUICKSTART_PYTHON.md)).

---

## Environment for local development

The simplest setup uses the bundled compose file:

```bash
docker compose up -d
```

Then leave all the `*_HOST` / `*_PORT` env vars at their defaults — `localhost` and the ports above match the compose file exactly.

For Kubernetes, point each `*_SERVICE_HOST` at the in-cluster Service DNS name (e.g. `remote-control-service.zequent.svc.cluster.local`).
