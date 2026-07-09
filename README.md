# Zequent Documentation

Last updated: 2026-07-08

Zequent provides SDKs, platform service images, and ready-to-run edge adapter images for connecting, monitoring, and controlling remote assets such as drones, docks, vehicles, and other edge devices.

This public documentation is for external developers and integration teams. It focuses on:

- using the Client SDKs from customer applications
- building custom edge adapters with the Edge SDKs
- running Zequent platform services from published container images
- deploying supported edge adapter images

## Start Here

| Goal | Documentation |
| --- | --- |
| Use Zequent from a Java application | [Java Client SDK Quickstart](client-sdk/QUICKSTART.md) |
| Use Zequent from a Python application | [Python Client SDK Quickstart](client-sdk/QUICKSTART_PYTHON.md) |
| Configure a customer application | [Client SDK Configuration](client-sdk/CONFIGURATION.md) |
| Build a custom Java edge adapter | [Java Edge SDK Quickstart](edge-sdk/edge-sdk-quickstart.md) |
| Build a custom Python edge adapter | [Python Edge SDK Quickstart](edge-sdk/edge-sdk-python-quickstart.md) |
| Configure an edge adapter | [Edge SDK Configuration](edge-sdk/edge-sdk-configuration.md) |
| Deploy the DJI adapter | [DJI Adapter Deployment](edge-sdk/edge-sdk-dji-adapter-deployment.md) |

## Customer Applications

Customer applications normally use the Client SDK and connect to the platform service endpoints exposed by your deployment.

| SDK | Main docs |
| --- | --- |
| Java Client SDK | [Quickstart](client-sdk/QUICKSTART.md), [Configuration](client-sdk/CONFIGURATION.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE.md) |
| Python Client SDK | [Quickstart](client-sdk/QUICKSTART_PYTHON.md), [Configuration](client-sdk/CONFIGURATION_PYTHON.md), [Customer Example](client-sdk/CUSTOMER_EXAMPLE_PYTHON.md) |

## Platform Service Images

Zequent platform services are run from published container images. Use versioned tags for production deployments.

| Component | Image | Default port | Customer-facing purpose |
| --- | --- | ---: | --- |
| Connector Service | `ghcr.io/zequent/connector-service:latest` | `8010` | Asset, mission, task, scheduler, organization, notification, and telemetry APIs |
| Remote Control Service | `ghcr.io/zequent/remote-control-service:latest` | `8002` | Direct asset commands such as takeoff, go-to, return-to-home, dock, camera, and manual-control commands |
| Live Data Service | `ghcr.io/zequent/live-data-service:latest` | `8003` | Live telemetry, task progress, and live-stream state |
| Mission Autonomy Service | `ghcr.io/zequent/mission-autonomy-service:latest` | `8004` | Mission and task scheduling/autonomy workflows |
| Admin Console API | `ghcr.io/zequent/admin-console-service:latest` | `8005` | HTTP/WebSocket API for the Admin Console |
| Admin Console UI | `ghcr.io/zequent/zqnt-admin-console-dashboard:latest` | `3001` | Browser UI for monitoring and operations |

## Edge Adapter Images

Use these adapter images when you want a ready-made integration. Use the Edge SDK when you need to build a custom adapter.

| Adapter | Image | Status | Notes |
| --- | --- | --- | --- |
| DJI | `ghcr.io/zequent/edge-dji:latest` | Available | DJI dock/drone integration |
| Betaflight | `ghcr.io/zequent/edge-betaflight:latest` | Available | Betaflight-compatible devices |
| MAVLink | `ghcr.io/zequent/edge-mavlink:latest` | Available | MAVLink-compatible vehicles |
| RNS | `ghcr.io/zequent/edge-rns:latest` | Available | RNS integration |
| Sapient | `ghcr.io/zequent/edge-sapient:latest` | Available | Sapient-compatible integration |
| AI Adapter | `ghcr.io/zequent/ai-adapter:latest` | Under development | Early-stage adapter; public functionality is not finalized yet |

## Deployment Configuration

Container deployments use one deployment-local `.env` file referenced by [docker-compose.customer.yml](docker-compose.customer.yml).

```yaml
services:
  connector-service:
    image: ghcr.io/zequent/connector-service:latest
    env_file:
      - .env
```

Do not commit `.env` files. Keep credentials, broker settings, stream URLs, and deployment-specific hostnames in your deployment environment or secret manager.

See [Client SDK Configuration](client-sdk/CONFIGURATION.md) for the public configuration model.

## Admin Console

The Admin Console is split into an API image and a UI image.

| Component | Default local URL |
| --- | --- |
| Admin Console UI | `http://localhost:3001` |
| Admin Console API | `http://localhost:8005` |

The Admin Console provides browser workflows for asset monitoring, telemetry, missions/tasks, remote control, live streams, adapter management, and service health.

## SDK Requirements

| SDK | Requirements |
| --- | --- |
| Java Client SDK / Java Edge SDK | Java 25, Maven 3.9+ recommended, Quarkus 3.x for Quarkus applications |
| Python Client SDK / Python Edge SDK | Python 3.12+, `uv` recommended, `grpc.aio` |

## Package Access

If you consume private Zequent packages, configure access to the relevant package registry before building your customer application or adapter.

For Maven packages, configure your `~/.m2/settings.xml` with a token that has package read access.

## Production Notes

- Run platform services and provided adapters from published container images.
- Use versioned image tags for production.
- Keep secrets outside Git.
- Use TLS and deployment-managed secrets for production environments.
- Use the Client SDKs for customer applications and the Edge SDKs for custom adapters.
