# Edge SDK -- Connector Service

The `ConnectorService` interface provides a high-level API for managing platform resources such as assets, missions, tasks, schedulers, organizations, and waypoints. It communicates with the Zequent Connector Service over gRPC and exposes all operations as `CompletableFuture`-based asynchronous calls.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Asset Management](#asset-management)
- [Mission Management](#mission-management)
- [Task Management](#task-management)
- [Scheduler Management](#scheduler-management)
- [Organization and Waypoints](#organization-and-waypoints)
- [Error Handling](#error-handling)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)

---

## Overview

The Connector Service is the platform's central registry for all entities: assets (docks, drones, vehicles), missions, tasks, schedulers, and organizations. From the edge adapter, you use the `ConnectorService` to:

- Register your asset when the adapter starts
- Update asset state as it changes
- Fetch mission and task definitions before executing them
- Retrieve waypoints for autonomous flight plans
- Look up organization information

The SDK provides a ready-to-use implementation (`ConnectorServiceImpl`) that handles all gRPC communication, Proto-to-DTO mapping, and error logging.

---

## How It Works

```
Your Adapter Code
      |
      v
ConnectorService (interface)
      |
      v
ConnectorServiceImpl  --(gRPC)-->  Connector Service (platform)
      |
      v
ProtoJsonMapper (MapStruct-based bidirectional mapping)
```

The `ConnectorServiceImpl` is created via a CDI producer method. It receives a `MutinyConnectorServiceGrpc.MutinyConnectorServiceStub` (the gRPC stub) and a `ProtoJsonMapper` (for converting between Proto messages and Java DTOs). You can inject `ConnectorService` directly into your adapter or any other CDI bean.

---

## Asset Management

### Register an Asset

Register your device with the platform on startup:

```java
import com.zequent.framework.utils.missionautonomy.dto.AssetDTO;

AssetDTO asset = new AssetDTO();
asset.setSn("YOUR_DEVICE_SN");
asset.setName("Dock Alpha");
asset.setAssetType("ASSET_TYPE_DOCK");
asset.setVendor("DJI");

connectorService.registerAsset(asset)
    .thenAccept(registered -> {
        log.info("Asset registered with ID: {}", registered.getId());
    })
    .exceptionally(err -> {
        log.error("Failed to register asset", err);
        return null;
    });
```

### Get Asset by Serial Number

```java
connectorService.getAssetBySn("YOUR_DEVICE_SN")
    .thenAccept(asset -> {
        log.info("Found asset: {} (ID: {})", asset.getName(), asset.getId());
    });
```

### Get Asset by ID

```java
connectorService.getAssetById("550e8400-e29b-41d4-a716-446655440000")
    .thenAccept(asset -> {
        log.info("Asset SN: {}", asset.getSn());
    });
```

### Update an Asset

```java
AssetDTO update = new AssetDTO();
update.setSn("YOUR_DEVICE_SN");
update.setName("Dock Alpha -- Updated");

connectorService.updateAsset("550e8400-e29b-41d4-a716-446655440000", update)
    .thenAccept(updated -> log.info("Asset updated"));
```

### Deregister an Asset

```java
connectorService.deRegisterAsset("550e8400-e29b-41d4-a716-446655440000")
    .thenAccept(success -> {
        if (success) {
            log.info("Asset deregistered");
        }
    });
```

### Get Sub-Asset by Serial Number

Retrieve a sub-asset (e.g., the drone paired to a dock):

```java
connectorService.getSubAssetBySn("YOUR_DEVICE_SNXXX")
    .thenAccept(subAsset -> {
        log.info("Sub-asset model: {}", subAsset.getModel());
    });
```

---

## Mission Management

### Get a Mission

```java
connectorService.getMissionById("mission-uuid")
    .thenAccept(mission -> {
        log.info("Mission: {}", mission.getName());
    });
```

### Create a Mission

```java
MissionDTO mission = new MissionDTO();
mission.setName("Perimeter Patrol");

connectorService.createMission(mission)
    .thenAccept(created -> {
        log.info("Mission created: {}", created.getId());
    });
```

### Update a Mission

```java
MissionDTO update = new MissionDTO();
update.setName("Perimeter Patrol v2");

connectorService.updateMission("mission-uuid", update)
    .thenAccept(updated -> log.info("Mission updated"));
```

### Delete a Mission

```java
connectorService.deleteMission("mission-uuid")
    .thenAccept(success -> {
        if (success) log.info("Mission deleted");
    });
```

---

## Task Management

Tasks represent individual flight plans or jobs within a mission.

### Get a Task

```java
connectorService.getTaskById("task-uuid")
    .thenAccept(task -> log.info("Task: {}", task));
```

### Get a Task by Flight ID

```java
connectorService.getTaskByFlightId("flight-id-123")
    .thenAccept(task -> log.info("Task for flight: {}", task));
```

### Create a Task

```java
TaskDTO task = new TaskDTO();
// populate task fields

connectorService.createTask(task)
    .thenAccept(created -> log.info("Task created: {}", created.getId()));
```

### Update a Task

```java
connectorService.updateTask("task-uuid", updatedTask)
    .thenAccept(updated -> log.info("Task updated"));
```

### Delete a Task

```java
connectorService.deleteTask("task-uuid")
    .thenAccept(success -> {
        if (success) log.info("Task deleted");
    });
```

---

## Scheduler Management

Schedulers define when and how often tasks or missions are executed.

### Get a Scheduler

```java
connectorService.getSchedulerById("scheduler-uuid")
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler));
```

### Create a Scheduler

```java
SchedulerDTO scheduler = new SchedulerDTO();
// populate scheduler fields

connectorService.createScheduler(scheduler)
    .thenAccept(created -> log.info("Scheduler created: {}", created.getId()));
```

### Update a Scheduler

```java
connectorService.updateScheduler("scheduler-uuid", updatedScheduler)
    .thenAccept(updated -> log.info("Scheduler updated"));
```

### Delete a Scheduler

```java
connectorService.deleteScheduler("scheduler-uuid")
    .thenAccept(success -> {
        if (success) log.info("Scheduler deleted");
    });
```

---

## Organization and Waypoints

### Get Organization

```java
connectorService.getOrganizationById("org-uuid")
    .thenAccept(org -> log.info("Organization: {}", org.getName()));
```

### Get Waypoints by Task ID

Retrieve the list of waypoints associated with a task, typically used to generate flight plans:

```java
connectorService.getWaypointsByTaskId("task-uuid")
    .thenAccept(waypoints -> {
        log.info("Waypoints count: {}", waypoints.size());
        for (WaypointDTO wp : waypoints) {
            log.info("  lat={}, lon={}, alt={}",
                wp.getLatitude(), wp.getLongitude(), wp.getAltitude());
        }
    });
```

---

## Error Handling

All `ConnectorServiceImpl` methods follow a consistent error handling pattern:

1. The gRPC response includes a `hasErrors` flag.
2. If `hasErrors` is `true`, the method logs the error and returns `null` (for entity methods) or `false` (for delete methods).
3. If the gRPC call itself fails (network error, timeout), the `CompletableFuture` completes exceptionally.

You should always handle both cases:

```java
connectorService.getAssetBySn("SOME_SN")
    .thenAccept(asset -> {
        if (asset == null) {
            log.warn("Asset not found or server error");
            return;
        }
        // use asset
    })
    .exceptionally(err -> {
        log.error("gRPC call failed", err);
        return null;
    });
```

---

## Configuration

The Connector Service gRPC client is configured in `application.properties`:

```properties
quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010
quarkus.grpc.clients.connector-service.keep-alive-without-calls=true
```

For container deployments, override with the service hostname:

```properties
quarkus.grpc.clients.connector-service.host=connector-service
quarkus.grpc.clients.connector-service.port=8080
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.

---

## Usage Examples

### Startup Registration Pattern

A common pattern is to register the asset on application startup and deregister on shutdown:

```java
import io.quarkus.runtime.StartupEvent;
import io.quarkus.runtime.ShutdownEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@ApplicationScoped
public class AssetRegistration {

    private final ConnectorService connectorService;
    private final EdgeClientConfig config;
    private String registeredAssetId;

    public AssetRegistration(ConnectorService connectorService, EdgeClientConfig config) {
        this.connectorService = connectorService;
        this.config = config;
    }

    void onStart(@Observes StartupEvent event) {
        AssetDTO asset = new AssetDTO();
        asset.setSn(config.sn());
        asset.setAssetType(config.assetType().name());
        asset.setVendor(config.assetVendor().name());

        connectorService.registerAsset(asset)
            .thenAccept(registered -> {
                registeredAssetId = registered.getId();
                log.info("Asset registered: {}", registeredAssetId);
            })
            .exceptionally(err -> {
                log.error("Asset registration failed", err);
                return null;
            });
    }

    void onStop(@Observes ShutdownEvent event) {
        if (registeredAssetId != null) {
            connectorService.deRegisterAsset(registeredAssetId).join();
            log.info("Asset deregistered");
        }
    }
}
```

### Fetching Waypoints for Task Execution

```java
@Override
public CompletableFuture<CommandResult> prepareTask(String taskId, String tid) {
    return connectorService.getWaypointsByTaskId(taskId)
        .thenCompose(waypoints -> {
            if (waypoints == null || waypoints.isEmpty()) {
                return CompletableFuture.completedFuture(
                    CommandResult.error("No waypoints for task", taskId)
                );
            }
            // Generate flight plan from waypoints and upload to device
            generateAndUploadFlightPlan(waypoints);
            return CompletableFuture.completedFuture(
                CommandResult.success("Task prepared with " + waypoints.size() + " waypoints", tid, taskId)
            );
        });
}
```

---

## API Summary

| Method | Return Type | Description |
|--------|-------------|-------------|
| `getAssetBySn(sn)` | `CompletableFuture<AssetDTO>` | Get asset by serial number |
| `getAssetById(id)` | `CompletableFuture<AssetDTO>` | Get asset by ID |
| `getSubAssetBySn(sn)` | `CompletableFuture<SubAssetDTO>` | Get sub-asset by serial number |
| `registerAsset(dto)` | `CompletableFuture<AssetDTO>` | Register a new asset |
| `updateAsset(id, dto)` | `CompletableFuture<AssetDTO>` | Update an existing asset |
| `deRegisterAsset(id)` | `CompletableFuture<Boolean>` | Deregister an asset |
| `getMissionById(id)` | `CompletableFuture<MissionDTO>` | Get mission by ID |
| `createMission(dto)` | `CompletableFuture<MissionDTO>` | Create a new mission |
| `updateMission(id, dto)` | `CompletableFuture<MissionDTO>` | Update an existing mission |
| `deleteMission(id)` | `CompletableFuture<Boolean>` | Delete a mission |
| `getTaskById(id)` | `CompletableFuture<TaskDTO>` | Get task by ID |
| `getTaskByFlightId(flightId)` | `CompletableFuture<TaskDTO>` | Get task by flight ID |
| `createTask(dto)` | `CompletableFuture<TaskDTO>` | Create a new task |
| `updateTask(id, dto)` | `CompletableFuture<TaskDTO>` | Update an existing task |
| `deleteTask(id)` | `CompletableFuture<Boolean>` | Delete a task |
| `getSchedulerById(id)` | `CompletableFuture<SchedulerDTO>` | Get scheduler by ID |
| `createScheduler(dto)` | `CompletableFuture<SchedulerDTO>` | Create a new scheduler |
| `updateScheduler(id, dto)` | `CompletableFuture<SchedulerDTO>` | Update an existing scheduler |
| `deleteScheduler(id)` | `CompletableFuture<Boolean>` | Delete a scheduler |
| `getOrganizationById(id)` | `CompletableFuture<OrganizationDTO>` | Get organization by ID |
| `getWaypointsByTaskId(id)` | `CompletableFuture<List<WaypointDTO>>` | Get waypoints for a task |
