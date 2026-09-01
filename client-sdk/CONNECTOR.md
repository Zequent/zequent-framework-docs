# Zequent Client SDK — Connector

`client.connector()` gives your application direct access to the platform's system of record: asset lookups, organization info, scheduler management, technical configuration, operational policies, and asset payloads. Most integrations only need a handful of these — asset lookup and scheduler management are the most common.

For Python, see [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md).

## Requests share a context

Every Connector request carries an optional `ConnectorRequestContext` for transaction correlation — you don't need to fill it in for normal use; the SDK generates a transaction ID automatically if you leave it unset.

```java
import com.zqnt.sdk.client.connector.domains.*;

var request = GetAssetBySnRequest.builder()
    .context(ConnectorRequestContext.builder().sn("YOUR_DEVICE_SN").build())
    .build();

client.connector().getAssetBySn(request)
    .thenAccept(response -> System.out.println("Asset: " + response));
```

## Assets

| Method | Purpose |
| --- | --- |
| `getAssetBySn(GetAssetBySnRequest)` | Look up an asset by its serial number |
| `getAssetById(GetAssetByIdRequest)` | Look up an asset by its platform ID |
| `getSubAssetBySn(GetSubAssetBySnRequest)` | Look up a sub-asset (e.g. the drone paired to a dock) |
| `registerAsset(RegisterAssetRequest)` | Register a new asset (normally done by an edge adapter, not a customer app) |
| `updateAsset(UpdateAssetRequest)` | Update asset metadata |
| `updateSubAsset(UpdateSubAssetRequest)` | Update sub-asset metadata |
| `deregisterAsset(DeregisterAssetRequest)` | Remove an asset |

```java
var request = GetAssetByIdRequest.builder()
    .assetId("550e8400-e29b-41d4-a716-446655440000")
    .build();

client.connector().getAssetById(request)
    .thenAccept(response -> System.out.println(response));
```

## Asset payloads

Payloads are arbitrary, versioned metadata blobs attached to an asset (e.g. flight-plan artifacts, calibration data).

| Method | Purpose |
| --- | --- |
| `upsertAssetPayload(UpsertAssetPayloadRequest)` | Create or update a payload |
| `listAssetPayloads(ListAssetPayloadsRequest)` | List payloads for an asset |
| `deleteAssetPayload(DeleteAssetPayloadRequest)` | Remove a payload |

## Organizations

```java
var request = GetOrganizationRequest.builder().build();
client.connector().getOrganization(request)
    .thenAccept(response -> System.out.println("Org: " + response));
```

## Missions

| Method | Purpose |
| --- | --- |
| `getMission(GetMissionRequest)` | Get a mission by ID |
| `createMission(CreateMissionRequest)` | Create a mission |
| `updateMission(UpdateMissionRequest)` | Update a mission |
| `deleteMission(DeleteMissionRequest)` | Delete a mission |
| `uploadMissionNfzZones(UploadMissionZonesRequest)` | Attach no-fly zones to a mission |

`MissionResponse`/`TaskResponse`/`WaypointsResponse` use `isSuccess()` + `getError()` rather than the `getHasErrors()` pattern above — check `success` before reading the response payload.

```java
import com.zqnt.utils.missionautonomy.domains.MissionDTO;
import com.zqnt.utils.mission.proto.MissionType;

var mission = MissionDTO.builder()
    .name("North Perimeter Patrol")
    .type(MissionType.MISSION_TYPE_PERIMETER_PATROL)
    .build();

var request = CreateMissionRequest.builder().mission(mission).build();
client.connector().createMission(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            log.warn("Create mission failed: {}", response.getError().getErrorMessage());
            return;
        }
        System.out.println("Mission created: " + response.getMissionId());
    });
```

### No-fly zones

A mission's no-fly zones (NFZ) tell the platform's route planner where it must route drones — and dock-return paths — around. A zone's `area` can be a polygon, bounding box, circle, or raw GeoJSON, controlled by `GeoAreaDTO.type`:

```java
import com.zqnt.utils.missionautonomy.domains.*;
import com.zqnt.utils.mission.proto.GeoAreaType;
import com.zqnt.utils.mission.proto.MissionZoneType;
import com.zqnt.utils.mission.proto.ZoneEnforcementType;

var zone = MissionZoneDTO.builder()
    .name("Substation exclusion")
    .type(MissionZoneType.MISSION_ZONE_TYPE_NO_FLY)
    .enforcementType(ZoneEnforcementType.ZONE_ENFORCEMENT_TYPE_HARD_BLOCK)
    .area(GeoAreaDTO.builder()
        .type(GeoAreaType.GEO_AREA_TYPE_CIRCLE)
        .center(GeoPointDTO.builder().latitude(52.520008).longitude(13.404954).build())
        .radiusMeters(150.0)
        .build())
    .active(true)
    .build();

var request = UploadMissionZonesRequest.builder()
    .missionId(missionId)
    .zones(List.of(zone))
    .replaceExisting(false)
    .build();

client.connector().uploadMissionNfzZones(request)
    .thenAccept(response -> System.out.println("Zones uploaded: " + response.isSuccess()));
```

`GeoAreaDTO.type` requirements: `GEO_AREA_TYPE_POLYGON` needs 3+ `vertices`; `GEO_AREA_TYPE_BOUNDING_BOX` needs exactly 2; `GEO_AREA_TYPE_CIRCLE` needs `center` + a positive `radiusMeters`; `GEO_AREA_TYPE_GEO_JSON` needs `geoJson`. `replaceExisting(true)` replaces the mission's whole zone set instead of appending.

## Tasks

| Method | Purpose |
| --- | --- |
| `getTask(GetTaskRequest)` | Get a task by ID |
| `getTaskByFlightId(GetTaskByFlightIdRequest)` | Get a task by its external flight ID |
| `createTask(CreateTaskRequest)` | Create a task |
| `updateTask(UpdateTaskRequest)` | Update a task |
| `deleteTask(DeleteTaskRequest)` | Delete a task |
| `getWaypointsByTaskId(GetWaypointsByTaskIdRequest)` | Get the resolved waypoint list for a task |

```java
var request = GetTaskByFlightIdRequest.builder()
    .flightId("FLIGHT-20260901-0001")
    .build();

client.connector().getTaskByFlightId(request)
    .thenAccept(response -> System.out.println("Task: " + response.getTaskId()));
```

`getTaskByFlightId` looks up a task by the external flight ID an edge adapter assigned it (`TaskDTO.externalTaskId`) — useful for correlating a vendor-side flight record back to its Zequent task without knowing the platform's task ID up front.

## Schedulers

Schedulers define when and how often a Skill or command runs (see [Applications & Skills](../concepts/applications-and-skills.md)).

| Method | Purpose |
| --- | --- |
| `getScheduler(GetSchedulerRequest)` | Get a scheduler by ID |
| `createScheduler(CreateSchedulerRequest)` | Create a scheduler |
| `createSchedulers(CreateSchedulersRequest)` | Create several schedulers in one call |
| `updateScheduler(UpdateSchedulerRequest)` | Update a scheduler |
| `deleteScheduler(DeleteSchedulerRequest)` | Delete a scheduler |
| `deleteSchedulers(DeleteSchedulersRequest)` | Delete several schedulers in one call |

```java
import com.zqnt.utils.missionautonomy.domains.SchedulerDTO;

var scheduler = SchedulerDTO.builder()
    .name("Nightly patrol")
    // .cron(...), .assetSn(...), etc. — see SchedulerDTO for the full field set
    .build();

var request = CreateSchedulerRequest.builder().scheduler(scheduler).build();
client.connector().createScheduler(request)
    .thenAccept(response -> System.out.println("Scheduler created: " + response));
```

## Technical configuration & policies

Read-only lookups useful when your application needs to mirror platform-side configuration or operational policy (e.g. no-fly zones, altitude limits) rather than hard-coding it.

| Method | Purpose |
| --- | --- |
| `getTechnicalConfigs(GetTechnicalConfigsRequest)` | Fetch technical configuration values |
| `getActivePoliciesByType(GetPoliciesRequest)` | Fetch active operational policies of a given type |
| `getAllActivePolicies(GetAllActivePoliciesRequest)` | Fetch every active operational policy |

## Skill Contracts

Every connected asset self-reports which commands it actually supports through its edge adapter — that's a **Skill Contract**. Customer applications typically only need to *read* this registry (e.g. to build a UI that only shows buttons for commands an asset actually supports); an edge adapter is what *writes* to it. See [Edge SDK — Connector](../edge-sdk/edge-sdk-connector.md#skill-contracts) for the adapter side.

| Method | Purpose |
| --- | --- |
| `listSkillContracts(SkillContractStatus, commandId)` | List known command contracts, optionally filtered |

```java
client.connector().listSkillContracts(SkillContractStatus.SKILL_CONTRACT_STATUS_ACTIVE, null)
    .thenAccept(contracts -> contracts.forEach(c -> System.out.println(c.getCommandId())));
```

## Error handling

Connector responses carry a `hasErrors` flag rather than throwing for expected business errors (not found, validation failure). Handle both:

```java
client.connector().getAssetBySn(request)
    .thenAccept(response -> {
        if (response.getHasErrors()) {
            log.warn("Asset lookup failed: {}", response.getError().getErrorMessage());
            return;
        }
        // use response
    })
    .exceptionally(err -> {
        log.error("Connector call failed", err);
        return null;
    });
```
