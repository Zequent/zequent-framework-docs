# Edge SDK -- Mission Autonomy Service

The `MissionAutonomyService` interface provides access to the platform's Mission Autonomy Service over gRPC. It allows edge adapters to create, update, and retrieve missions, tasks, and schedulers programmatically.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [MissionAutonomyService Interface](#missionautonomyservice-interface)
- [Missions](#missions)
- [Tasks](#tasks)
- [Schedulers](#schedulers)
- [Configuration](#configuration)
- [Usage in an Adapter](#usage-in-an-adapter)

---

## Overview

The Mission Autonomy Service is responsible for the full lifecycle of missions within the Zequent platform: creation, scheduling, execution tracking, and completion. From the edge side, you interact with this service primarily to:

- Retrieve mission and task definitions that the platform has scheduled for this device.
- Report mission status or create missions programmatically.
- Look up task details (including waypoints via the Connector Service) before executing them on the hardware.
- Retrieve scheduler information for timed or recurring missions.

The Edge SDK provides the `MissionAutonomyService` interface and its default implementation `MissionAutonomyServiceImpl`, which handles gRPC communication with the `MutinyMissionAutonomyServiceGrpc` stub.

---

## How It Works

```
Your Adapter Code
      |
      v
MissionAutonomyService (interface)
      |
      v
MissionAutonomyServiceImpl  --(gRPC)-->  Mission Autonomy Service (platform)
```

The implementation is created via a CDI producer that injects the `MutinyMissionAutonomyServiceGrpc.MutinyMissionAutonomyServiceStub`. All methods return Mutiny `Uni` types for reactive composition.

---

## MissionAutonomyService Interface

```java
public interface MissionAutonomyService {

    Uni<MissionData> createMission(CreateMissionRequest createMissionRequest);

    Uni<MissionData> updateMission(UpdateMissionRequest updateMissionRequest);

    Uni<MissionData> getMission(GetMissionRequest getRequest);

    Uni<TaskProtoDTO> getTask(GetTaskRequest getTaskRequest);

    Uni<TaskProtoDTO> getTaskByFlightId(GetTaskRequest getTaskRequest);

    Uni<SchedulerProtoDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

All request types (`CreateMissionRequest`, `UpdateMissionRequest`, `GetMissionRequest`, `GetTaskRequest`, `GetSchedulerRequest`) are Protobuf-generated classes from the `mission-autonomy.proto` definition.

---

## Missions

### Get a Mission

Retrieve mission details by ID:

```java
import com.zequent.framework.services.mission.proto.GetMissionRequest;

GetMissionRequest request = GetMissionRequest.newBuilder()
    .setMissionId("mission-uuid")
    .build();

missionAutonomyService.getMission(request)
    .subscribe().with(
        mission -> log.info("Mission: {} (ID: {})", mission.getName(), mission.getId()),
        err -> log.error("Failed to get mission", err)
    );
```

### Create a Mission

```java
import com.zequent.framework.services.mission.proto.CreateMissionRequest;

CreateMissionRequest request = CreateMissionRequest.newBuilder()
    .setName("Perimeter Patrol")
    // set other fields as needed
    .build();

missionAutonomyService.createMission(request)
    .subscribe().with(
        mission -> log.info("Mission created: {}", mission.getId()),
        err -> log.error("Failed to create mission", err)
    );
```

### Update a Mission

```java
import com.zequent.framework.services.mission.proto.UpdateMissionRequest;

UpdateMissionRequest request = UpdateMissionRequest.newBuilder()
    .setMissionId("mission-uuid")
    .setName("Perimeter Patrol v2")
    .build();

missionAutonomyService.updateMission(request)
    .subscribe().with(
        mission -> log.info("Mission updated: {}", mission.getName()),
        err -> log.error("Failed to update mission", err)
    );
```

### MissionData Model

The `MissionData` POJO contains:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `UUID` | Unique mission identifier |
| `name` | `String` | Mission name |

---

## Tasks

Tasks represent individual executable units within a mission, such as a flight plan or patrol route.

### Get a Task by ID

```java
import com.zequent.framework.services.mission.proto.GetTaskRequest;

GetTaskRequest request = GetTaskRequest.newBuilder()
    .setTaskId("task-uuid")
    .build();

missionAutonomyService.getTask(request)
    .subscribe().with(
        task -> log.info("Task retrieved: {}", task),
        err -> log.error("Failed to get task", err)
    );
```

### Get a Task by Flight ID

When a task is associated with a flight execution, you can look it up by the flight identifier:

```java
GetTaskRequest request = GetTaskRequest.newBuilder()
    .setTaskId("flight-id-123")
    .build();

missionAutonomyService.getTaskByFlightId(request)
    .subscribe().with(
        task -> log.info("Task for flight: {}", task),
        err -> log.error("Failed to get task by flight ID", err)
    );
```

---

## Schedulers

Schedulers define the timing and recurrence of mission or task execution.

### Get a Scheduler

```java
import com.zequent.framework.services.mission.proto.GetSchedulerRequest;

GetSchedulerRequest request = GetSchedulerRequest.newBuilder()
    .setSchedulerId("scheduler-uuid")
    .build();

missionAutonomyService.getScheduler(request)
    .subscribe().with(
        scheduler -> log.info("Scheduler: {}", scheduler),
        err -> log.error("Failed to get scheduler", err)
    );
```

---

## Configuration

The Mission Autonomy Service gRPC client can be configured if your adapter needs to connect to it. In most setups, the mission autonomy interaction happens through the Connector Service, but direct access is available when needed.

Add to `application.properties` if not already present:

```properties
quarkus.grpc.clients.mission-autonomy-service.host=localhost
quarkus.grpc.clients.mission-autonomy-service.port=9092
quarkus.grpc.clients.mission-autonomy-service.keep-alive-without-calls=true
```

For container deployments:

```properties
quarkus.grpc.clients.mission-autonomy-service.host=mission-autonomy-service
```

---

## Usage in an Adapter

A typical pattern is to use the Mission Autonomy Service together with the Connector Service when preparing or executing tasks. For example, when a `prepareTask` command comes in from the platform:

```java
@ApplicationScoped
public class MyAdapter implements EdgeAdapterService {

    private final MissionAutonomyService missionService;
    private final ConnectorService connectorService;

    public MyAdapter(MissionAutonomyService missionService, ConnectorService connectorService) {
        this.missionService = missionService;
        this.connectorService = connectorService;
    }

    @Override
    public CompletableFuture<CommandResult> prepareTask(String taskId, String tid) {
        // 1. Fetch task details from Mission Autonomy
        return missionService.getTask(
                GetTaskRequest.newBuilder().setTaskId(taskId).build()
            )
            .chain(task -> {
                // 2. Fetch waypoints from Connector
                return Uni.createFrom().completionStage(
                    connectorService.getWaypointsByTaskId(taskId)
                );
            })
            .map(waypoints -> {
                if (waypoints == null || waypoints.isEmpty()) {
                    return CommandResult.error("No waypoints found for task", taskId);
                }
                // 3. Generate flight plan and upload to device
                generateFlightPlan(waypoints);
                return CommandResult.success("Task prepared", tid, taskId);
            })
            .subscribeAsCompletionStage();
    }

    private void generateFlightPlan(List<WaypointDTO> waypoints) {
        // Implementation depends on your device vendor
    }
}
```

This pattern -- fetch task, fetch waypoints, generate plan -- is used by the reference DJI adapter implementation and can be adapted for any vendor.

---

## API Summary

| Method | Return Type | Description |
|--------|-------------|-------------|
| `createMission(CreateMissionRequest)` | `Uni<MissionData>` | Create a new mission |
| `updateMission(UpdateMissionRequest)` | `Uni<MissionData>` | Update an existing mission |
| `getMission(GetMissionRequest)` | `Uni<MissionData>` | Get mission by ID |
| `getTask(GetTaskRequest)` | `Uni<TaskProtoDTO>` | Get task by ID |
| `getTaskByFlightId(GetTaskRequest)` | `Uni<TaskProtoDTO>` | Get task by flight ID |
| `getScheduler(GetSchedulerRequest)` | `Uni<SchedulerProtoDTO>` | Get scheduler by ID |
