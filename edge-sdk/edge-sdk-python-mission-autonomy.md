# Edge SDK (Python) — Mission Autonomy

The Python Edge SDK exposes mission, task, and scheduler operations through the same gRPC contract as the Java SDK. Most edge adapters interact with these only on the **receiving** side — the platform sends `prepare_task` / `start_task` / `stop_task` to your adapter; you don't usually need to call the Mission Autonomy service directly.

If you do need to author missions/tasks/schedulers programmatically from an edge process, use the **Client SDK** (`zqnt-client-sdk`) — it wraps the same gRPC service and has the right ergonomics for "push to the platform" use cases.

For Java, see [edge-sdk-mission-autonomy.md](edge-sdk-mission-autonomy.md).

---

## Receiving tasks (typical edge case)

The platform calls these methods on your `EdgeAdapter`:

```python
from edge_sdk import EdgeAdapter, EdgeResponse, Task

class MyAdapter(EdgeAdapter):

    async def prepare_task(self, ctx, task: Task) -> EdgeResponse:
        # validate task config, pre-route waypoints, reserve resources
        if not self._can_handle(task):
            return EdgeResponse.error(ctx.tid, ctx.sn, "Unsupported task type")
        self._pending[task.id] = task
        return EdgeResponse.success(ctx.tid, ctx.sn, "Task prepared")

    async def start_task(self, ctx, task: Task) -> EdgeResponse:
        # begin execution; return immediately, push progress via telemetry
        self._executor.submit(task)
        return EdgeResponse.success(ctx.tid, ctx.sn, "Task started")

    async def stop_task(self, ctx, task_id: str) -> EdgeResponse:
        await self._executor.cancel(task_id)
        return EdgeResponse.success(ctx.tid, ctx.sn, "Task stopped")
```

---

## Task model

`Task` (and the various `*TaskConfig` variants) live in `edge_sdk.models.task`:

| Class                    | Use                                                 |
|--------------------------|-----------------------------------------------------|
| `Task`                   | Top-level container with id, type, status, config  |
| `WaypointTaskConfig`     | Fly through ordered `Waypoint` list                 |
| `DetectTaskConfig`       | AI detection on a flight path                       |
| `AreaMappingTaskConfig`  | Survey an area defined by `AreaVertex` polygon      |
| `PoiTaskConfig`          | Point-of-interest orbit                             |
| `FollowTaskConfig`       | Follow a moving subject                             |
| `TrackTaskConfig`        | Track a detected object                             |

```python
from edge_sdk import Task, TaskType, WaypointTaskConfig, Waypoint

# Inspect the incoming task in start_task:
async def start_task(self, ctx, task: Task) -> EdgeResponse:
    if task.type == TaskType.WAYPOINT:
        cfg: WaypointTaskConfig = task.waypoint_config
        for wp in cfg.waypoints:
            await self._fly_to(wp.latitude, wp.longitude, wp.altitude)
    return EdgeResponse.success(ctx.tid, ctx.sn, "Task complete")
```

---

## Reporting progress

Progress and detection results flow back to the platform via `TelemetryPublisher`, **not** as task RPC return values:

```python
from edge_sdk import CommandProgress, DetectionResponse

await pub.publish_progress(
    task_id=task.id,
    progress=CommandProgress(percent=42, status="surveying"),
)

await pub.publish_detection(
    DetectionResponse(task_id=task.id, results=[...]),
)
```

This lets long-running tasks stream updates without holding open RPC calls.

---

## Mission and scheduler CRUD (rare for edge)

If you must, do it from the edge process via the Client SDK:

```python
from client_sdk import ZequentClient
from client_sdk.models import MissionDTO, TaskDTO, SchedulerDTO

async with ZequentClient.from_env() as client:
    mission = await client.mission_autonomy.create_mission(MissionDTO(...))
    task = await client.mission_autonomy.create_task(TaskDTO(mission_id=mission.id, ...))
    sched = await client.mission_autonomy.create_scheduler(SchedulerDTO(task_id=task.id, ...))
```

This is intentionally **not** in the Edge SDK — keeping the Edge SDK focused on the device-facing contract avoids tight coupling with platform-side data models.

---

## Best practices

- **Validate in `prepare_task`**, return an error there if you can't handle the task. Don't accept and then fail in `start_task`.
- **Make `start_task` non-blocking.** Schedule the work and return success immediately. Use telemetry / progress pushes to report state.
- **Idempotent `stop_task`.** Cancelling a task that's already finished must be a no-op.
- **Persist `task.id`** if you need to recover after a restart; the platform may re-issue a `start_task` for a task you already started.
