# Edge SDK (Python) — Quickstart Guide

This guide walks you through creating a new edge adapter project from scratch using the Zequent Edge SDK for Python. By the end, you will have a running Python application that receives commands from the platform and pushes telemetry data.

## Prerequisites

- Python **3.12+**
- [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`
- Docker or Podman (for running platform services)
- GitHub account with access to the Zequent packages repository

---

## Step 1: Create a new project

```bash
mkdir my-edge-adapter && cd my-edge-adapter
uv init --python 3.12
```

This creates a `pyproject.toml` and a `.venv/` managed by `uv`.

---

## Step 2: Add the Edge SDK dependency

```bash
uv add edge-python-sdk
```

If pulling from a private GitHub repository, configure `[tool.uv.sources]` in `pyproject.toml`:

```toml
[tool.uv.sources]
edge-python-sdk = { git = "https://github.com/Zequent/zqnt-edge-sdk-python", rev = "v1.0.0" }
```

and provide a token via `GITHUB_TOKEN` when running `uv sync`.

---

## Step 3: Configure the adapter

Create a `.env` file (or export environment variables in your shell):

```bash
# Edge identity
ZEQUENT_EDGE_ENDPOINT=localhost:9001
ZEQUENT_EDGE_SN=YOUR_DEVICE_SERIAL_NUMBER
ZEQUENT_EDGE_ASSET_TYPE=ASSET_TYPE_DOCK
ZEQUENT_EDGE_ASSET_VENDOR=DJI

# Platform services
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003
CONNECTOR_SERVICE_HOST=localhost
CONNECTOR_SERVICE_PORT=8010
```

The SDK reads these via `os.environ` when you call the corresponding factory helpers.

---

## Step 4: Implement the `EdgeAdapter`

Create `my_edge_adapter/adapter.py`. You only override the commands your hardware supports; everything else returns `EdgeResponse.not_supported(...)` automatically.

```python
import logging
from edge_sdk import (
    EdgeAdapter, EdgeResponse,
    Capabilities, AssetType,
    RequestContext, Coordinates,
    ReturnToHomeRequest,
)

log = logging.getLogger(__name__)


class MyDeviceAdapter(EdgeAdapter):

    async def get_capabilities(self, sn: str, asset_id: str | None) -> Capabilities:
        # Auto-discovers which commands you implemented.
        return self._auto_capabilities(sn, AssetType.DOCK)

    async def take_off(self, ctx: RequestContext, coordinates: Coordinates) -> EdgeResponse:
        log.info("Takeoff requested for SN: %s", ctx.sn)
        # call your hardware's takeoff API here
        ok = True
        if ok:
            return EdgeResponse.success(ctx.tid, ctx.sn, "Takeoff initiated")
        return EdgeResponse.error(ctx.tid, ctx.sn, "Takeoff failed")

    async def return_to_home(
        self, ctx: RequestContext, request: ReturnToHomeRequest
    ) -> EdgeResponse:
        log.info("RTH for SN: %s", ctx.sn)
        return EdgeResponse.success(ctx.tid, ctx.sn, "Returning to home")

    async def open_cover(self, ctx: RequestContext) -> EdgeResponse:
        log.info("Opening cover for SN: %s", ctx.sn)
        return EdgeResponse.success(ctx.tid, ctx.sn, "Cover opening")
```

---

## Step 5: Push telemetry data (optional)

If your device produces telemetry, use `TelemetryPublisher`:

```python
import asyncio
from datetime import datetime, timezone
from edge_sdk import TelemetryPublisher, AssetTelemetry, AssetPositionState


async def telemetry_loop(sn: str):
    pub = TelemetryPublisher(host="localhost", port=8003, sn=sn)
    await pub.connect()
    try:
        while True:
            await pub.publish_asset_telemetry(
                AssetTelemetry(
                    id=sn,
                    timestamp=datetime.now(tz=timezone.utc),
                    position=AssetPositionState(
                        latitude=47.3769, longitude=8.5417, altitude=450.0,
                    ),
                    battery_percentage=87,
                )
            )
            await asyncio.sleep(1.0)
    finally:
        await pub.close()
```

---

## Step 6: Run the adapter

`my_edge_adapter/__main__.py`:

```python
import asyncio
import logging

from edge_sdk import EdgeServer
from .adapter import MyDeviceAdapter


async def main():
    logging.basicConfig(level=logging.INFO)
    server = EdgeServer(adapter=MyDeviceAdapter(), port=9001)
    await server.serve()


if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
uv run python -m my_edge_adapter
```

You should see:

```
INFO  edge_sdk.server.edge_server  EdgeServer listening on 0.0.0.0:9001
```

---

## Step 7: Verify against the platform

With the platform services running from the published container images (see [Zequent Documentation](../README.md)):

1. The platform discovers your adapter via the configured endpoint.
2. Sending a takeoff command from a Client SDK (Java or Python) lands on your `take_off` method.
3. Telemetry pushed via `TelemetryPublisher` is visible through the Live Data service.

---

## Next steps

- [Configuration reference](edge-sdk-python-configuration.md) — every env var the SDK reads
- [Adapter reference](edge-sdk-python-adapter.md) — full list of overridable methods
- [Live Data reference](edge-sdk-python-live-data.md) — telemetry model details
- [Connector reference](edge-sdk-python-connector.md) — asset registration helpers
