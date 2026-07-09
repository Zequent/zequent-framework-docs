# Zequent Edge SDK (Python)

The Edge SDK is used to build custom edge adapters that connect physical assets such as drones, docks, vehicles, and other remote devices to Zequent. A custom adapter exposes a consistent command and telemetry interface to the platform and to Client SDK consumers.

This is the **Python** edition of the Edge SDK. For Java, see [edge-sdk-overview.md](edge-sdk-overview.md).

## Tech Specs

| Requirement   | Version                       |
|---------------|-------------------------------|
| Python        | 3.12+                         |
| Package mgr   | `uv` (recommended) or `pip`   |
| gRPC runtime  | `grpcio` / `grpcio-tools`     |
| Async runtime | `asyncio` (`grpc.aio`)        |

## Overview

The Edge SDK sits between the physical device and the Zequent platform services. An edge adapter is a standalone Python application that depends on the `edge-python-sdk` package and provides concrete implementations for the commands relevant to its hardware.

```
+-------------------+     gRPC     +------------------------+
| Zequent Platform  | <----------> |  Your Edge Adapter     |
| (container images)|              |  (Python application)  |
+-------------------+              |  - EdgeAdapter subclass|
                                   |  - EdgeServer          |
                                   |  - TelemetryPublisher  |
                                   +------------------------+
                                              |
                                              v
                                       Physical hardware
```

## Core Concepts

### `EdgeAdapter`

An `abc.ABC` you subclass to provide hardware-specific behaviour. Only `get_capabilities` is abstract — every other method has a default implementation that returns `EdgeResponse.not_supported(...)`. Override only the commands your hardware supports.

### `EdgeServer`

The async gRPC server that hosts your `EdgeAdapter`. Routes incoming RPC calls to your overridden methods, converts protobuf messages to Python dataclasses, and reports unimplemented methods correctly.

### `TelemetryPublisher`

Manages a persistent gRPC connection to the Live Data service. Lets you push asset and sub-asset telemetry using strongly-typed dataclasses (`AssetTelemetry`, `SubAssetTelemetry`).

### `ConnectorClient`

Talks to the platform's Connector Service over gRPC for asset registration / lookup and other CRUD operations.

## Available Documentation

| Document                                                                  | Description                                                          |
|---------------------------------------------------------------------------|----------------------------------------------------------------------|
| [Quickstart](edge-sdk-python-quickstart.md)                               | Get a new Python edge adapter project running in minutes             |
| [Configuration](edge-sdk-python-configuration.md)                         | All configuration knobs and environment variable mappings            |
| [Edge Adapter](edge-sdk-python-adapter.md)                                | Implementing the `EdgeAdapter` base class                            |
| [Live Data](edge-sdk-python-live-data.md)                                 | Producing telemetry data streams from your adapter                   |
| [Connector](edge-sdk-python-connector.md)                                 | Asset and resource management via the Connector Service              |
| [Mission Autonomy](edge-sdk-python-mission-autonomy.md)                   | Working with missions, tasks, and schedulers                         |
| [Models Reference](edge-sdk-python-models.md)                             | Request, response, and telemetry data model reference                |

## Quick Start

Install the Edge SDK:

```bash
uv add edge-python-sdk
# or
pip install edge-python-sdk
```

Configure your edge via environment variables:

```bash
export ZEQUENT_EDGE_ENDPOINT=localhost:9001
export ZEQUENT_EDGE_SN=YOUR_DEVICE_SERIAL_NUMBER
export ZEQUENT_EDGE_ASSET_TYPE=ASSET_TYPE_DOCK
export ZEQUENT_EDGE_ASSET_VENDOR=DJI
```

Implement the adapter:

```python
import asyncio
from edge_sdk import (
    EdgeAdapter, EdgeServer, EdgeResponse,
    Capabilities, AssetType, RequestContext, Coordinates,
)


class MyEdgeAdapter(EdgeAdapter):

    async def get_capabilities(self, sn: str, asset_id: str | None) -> Capabilities:
        return self._auto_capabilities(sn, AssetType.DOCK)

    async def take_off(self, ctx: RequestContext, coordinates: Coordinates) -> EdgeResponse:
        # call your hardware-specific takeoff
        return EdgeResponse.success(ctx.tid, ctx.sn, "Takeoff initiated")


async def main():
    server = EdgeServer(adapter=MyEdgeAdapter(), port=9001)
    await server.serve()


asyncio.run(main())
```

For a complete walkthrough, see the [Quickstart Guide](edge-sdk-python-quickstart.md).
