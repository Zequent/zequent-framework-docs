# Zequent Framework Edge SDK

The Edge SDK provides the foundation for building Edge Adapters that connect physical assets (drones, docks, vehicles, and other remote devices) to the Zequent Framework ecosystem. By implementing the interfaces defined in this SDK, each vendor-specific edge adapter exposes a uniform set of commands and data streams that the platform and its Client SDK consumers can use without any knowledge of the underlying hardware.

## Tech Specs

| Requirement | Version |
|-------------|---------|
| Java        | 25      |
| Maven       | 3.9.9+  |
| Quarkus     | 3.x     |
| gRPC        | via Quarkus gRPC extension |

## Architecture Overview

The Edge SDK sits between the physical device and the Zequent platform services. An edge adapter is a standalone Quarkus application that depends on the Edge SDK library and provides concrete implementations for the commands relevant to its hardware.


## Core Concepts

### EdgeClient

The central entry point of the SDK. It is a CDI-managed bean that gives you access to the configured serial number, edge configuration, and the `EdgeAdapterService` instance.

### EdgeAdapterService

The primary interface that every edge adapter must implement. It declares commands for flight control, dock management, camera operations, manual control, live streaming, task execution, and more. All methods come with a default `NOT_IMPLEMENTED` return so you only need to override what your hardware supports.

### ConnectorService

Provides CRUD operations against the platform's Connector Service over gRPC. Use it to register and manage assets, missions, tasks, schedulers, waypoints, and organizations.

### LiveDataService

Manages persistent gRPC telemetry streams. It lets you push asset and sub-asset telemetry data to the Live Data Service using either the POJO-based API or the raw Proto-based API.

### MissionAutonomyService

Communicates with the Mission Autonomy Service over gRPC to create, update, and retrieve missions, tasks, and schedulers.

## Available Documentation

| Document | Description |
|----------|-------------|
| [Quickstart](edge-sdk-quickstart.md) | Get a new edge adapter project up and running in minutes |
| [Configuration](edge-sdk-configuration.md) | All configuration properties and environment variable mappings |
| [Edge Adapter](edge-sdk-adapter.md) | Implementing the EdgeAdapterService interface |
| [Live Data](edge-sdk-live-data.md) | Producing telemetry data streams |
| [Connector](edge-sdk-connector.md) | Asset and resource management via the Connector Service |
| [Mission Autonomy](edge-sdk-mission-autonomy.md) | Working with missions, tasks, and schedulers |
| [Models Reference](edge-sdk-models.md) | Request, response, and telemetry data model reference |

## Quick Start

Add the Edge SDK dependency to your project:

```xml
<dependency>
  <groupId>com.zqnt.sdk</groupId>
  <artifactId>edge-sdk</artifactId>
  <version>1.0.0</version>
</dependency>
```

Configure your edge in `application.properties`:

```properties
zequent.edge.endpoint=localhost:9001
zequent.edge.sn=YOUR_DEVICE_SERIAL_NUMBER
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=DJI
```

Implement the adapter interface:

```java
@ApplicationScoped
public class MyEdgeAdapter implements EdgeAdapterService {

    @Override
    public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
        // your hardware-specific takeoff logic
        return CompletableFuture.completedFuture(
            CommandResult.success("Takeoff initiated", request.getSn())
        );
    }

    // override only the commands your device supports
}
```

Run the application:

```bash
./mvnw quarkus:dev
```

For a complete walkthrough, see the [Quickstart Guide](edge-sdk-quickstart.md).
