# Edge SDK -- Mission Autonomy Service

`MissionAutonomyService` is a small, focused interface: it lets an edge adapter look up a **scheduler** definition directly from the Mission Autonomy service. Everything else related to running automated behavior on an asset — receiving task lifecycle calls, reporting progress, executing Applications/Skills — happens through other parts of the SDK, described below.

## Table of Contents

- [Overview](#overview)
- [MissionAutonomyService Interface](#missionautonomyservice-interface)
- [Where task execution actually happens](#where-task-execution-actually-happens)
- [Configuration](#configuration)

---

## Overview

Most edge adapters never call `MissionAutonomyService` directly. The platform drives task execution by calling *into* your adapter (`prepareTask` / `startTask` / `pauseTask` / `resumeTask` / `stopTask` / `cancelExecution` on `EdgeAdapterService`), and your adapter reports progress back over the Live Data connection — it doesn't poll or manage missions itself.

`MissionAutonomyService` exists for the one case where an adapter needs scheduler metadata directly:

```java
public interface MissionAutonomyService {
    CompletableFuture<SchedulerDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

```java
import com.zqnt.utils.mission.proto.GetSchedulerRequest;

GetSchedulerRequest request = GetSchedulerRequest.newBuilder()
    .setSchedulerId("scheduler-uuid")
    .build();

missionAutonomyService.getScheduler(request)
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler))
    .exceptionally(err -> {
        log.error("Failed to get scheduler", err);
        return null;
    });
```

Scheduler CRUD (create/update/delete) is available through `ConnectorService` instead — see [Connector](edge-sdk-connector.md#schedulers).

---

## Where task execution actually happens

| Concern | Where it lives |
| --- | --- |
| Receiving `prepareTask`/`startTask`/`pauseTask`/`resumeTask`/`stopTask`/`cancelExecution` calls | `EdgeAdapterService` — see [Edge Adapter](edge-sdk-adapter.md#task-execution) |
| Reporting progress/telemetry while a task runs | `LiveDataService` — see [Live Data](edge-sdk-live-data.md) |
| Declaring which commands your adapter supports | `ConnectorService`'s Skill Contract registry — see [Connector](edge-sdk-connector.md#skill-contracts) |
| Authoring, deploying, and triggering multi-step Applications/Skills | The **Client SDK**, used by customer applications — see [Applications & Skills](../concepts/applications-and-skills.md) |

A typical `prepareTask` implementation looks up whatever it needs (e.g. a stored flight plan) via `ConnectorService`'s asset payload methods, rather than through `MissionAutonomyService`:

```java
@Override
public CompletableFuture<CommandResult> prepareTask(String taskId, String tid) {
    // Fetch whatever your adapter needs to execute this task — e.g. a previously
    // uploaded flight plan stored as an asset payload — then stage it on the device.
    return CompletableFuture.completedFuture(
        CommandResult.success("Task prepared", tid, taskId)
    );
}
```

---

## Configuration

```properties
quarkus.grpc.clients.mission-autonomy-service.host=localhost
quarkus.grpc.clients.mission-autonomy-service.port=8004
quarkus.grpc.clients.mission-autonomy-service.keep-alive-without-calls=true
```

For container deployments:

```properties
quarkus.grpc.clients.mission-autonomy-service.host=mission-autonomy-service
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.
