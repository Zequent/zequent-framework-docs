# Edge SDK -- Connector Service

The `ConnectorService` interface gives an edge adapter access to the platform's asset registry and Skill Contract registry over gRPC. It covers what an adapter itself needs — registering and updating its own asset(s), looking up schedulers and organization info, and reporting which commands it supports — not general mission/task management (that belongs to the **Client SDK**, used by customer applications; see [Applications & Skills](../concepts/applications-and-skills.md)).

## Table of Contents

- [Overview](#overview)
- [Asset Management](#asset-management)
- [Asset Payloads](#asset-payloads)
- [Schedulers](#schedulers)
- [Organization](#organization)
- [Skill Contracts](#skill-contracts)
- [Error Handling](#error-handling)
- [Configuration](#configuration)
- [Usage Examples](#usage-examples)
- [API Summary](#api-summary)

---

## Overview

From the edge adapter, you use `ConnectorService` to:

- Register your asset when the adapter starts, and deregister it on shutdown.
- Update asset state as it changes.
- Fetch a scheduler's definition and the organization it belongs to.
- Store and retrieve asset payloads (arbitrary versioned metadata blobs, e.g. calibration data).
- Report which commands your adapter currently supports, via the **Skill Contract** registry — this is what lets the Admin Console's graph editor validate a Skill against real device capability while it's being authored.

The SDK provides a ready-to-use implementation (`ConnectorServiceImpl`) that handles gRPC communication and Proto-to-DTO mapping.

---

## Asset Management

### Register an Asset

```java
import com.zqnt.utils.asset.domains.AssetDTO;

AssetDTO asset = new AssetDTO();
asset.setSn("YOUR_DEVICE_SN");
asset.setName("Dock Alpha");
asset.setAssetType("ASSET_TYPE_DOCK");
asset.setVendor("DJI");

connectorService.registerAsset(asset)
    .thenAccept(registered -> log.info("Asset registered with ID: {}", registered.getId()))
    .exceptionally(err -> {
        log.error("Failed to register asset", err);
        return null;
    });
```

### Get Asset by Serial Number / ID

```java
connectorService.getAssetBySn("YOUR_DEVICE_SN")
    .thenAccept(asset -> log.info("Found asset: {} (ID: {})", asset.getName(), asset.getId()));

connectorService.getAssetById("550e8400-e29b-41d4-a716-446655440000")
    .thenAccept(asset -> log.info("Asset SN: {}", asset.getSn()));
```

### Get Sub-Asset by Serial Number

Retrieve a sub-asset (e.g. the drone paired to a dock):

```java
connectorService.getSubAssetBySn("YOUR_DEVICE_SNXXX")
    .thenAccept(subAsset -> log.info("Sub-asset model: {}", subAsset.getModel()));
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
        if (success) log.info("Asset deregistered");
    });
```

---

## Asset Payloads

Store arbitrary metadata alongside an asset or sub-asset — for example, a generated flight-plan artifact or calibration data.

```java
connectorService.upsertAssetPayload("YOUR_DEVICE_SN", null, payloadDTO)
    .thenAccept(saved -> log.info("Payload stored: {}", saved.getId()));
```

---

## Schedulers

Schedulers define when and how often a Skill or command runs — see [Applications & Skills](../concepts/applications-and-skills.md).

```java
connectorService.getSchedulerById("scheduler-uuid")
    .thenAccept(scheduler -> log.info("Scheduler: {}", scheduler));

connectorService.createScheduler(schedulerDTO)
    .thenAccept(created -> log.info("Scheduler created: {}", created.getId()));

connectorService.updateScheduler("scheduler-uuid", updatedScheduler)
    .thenAccept(updated -> log.info("Scheduler updated"));

connectorService.deleteScheduler("scheduler-uuid")
    .thenAccept(success -> {
        if (success) log.info("Scheduler deleted");
    });
```

---

## Organization

```java
connectorService.getOrganizationById("org-uuid")
    .thenAccept(org -> log.info("Organization: {}", org.getName()));
```

---

## Skill Contracts

A Skill Contract is your adapter's self-reported declaration of which commands it supports, with their parameter schema. Reporting this accurately is what lets the Admin Console warn an operator at Skill-authoring time — "this asset doesn't support ChangeLens" — instead of failing at execution time.

```java
import com.zqnt.utils.connector.proto.SkillContractProtoDTO;
import com.zqnt.utils.connector.proto.SkillContractStatus;

connectorService.observeSkillContract(contract)
    .thenAccept(observed -> log.info("Contract observed: {}", observed.getCommandId()));

connectorService.listSkillContracts(SkillContractStatus.SKILL_CONTRACT_STATUS_ACTIVE, null)
    .thenAccept(contracts -> log.info("Known contracts: {}", contracts.size()));

connectorService.setSkillContractStatus("contract-id", SkillContractStatus.SKILL_CONTRACT_STATUS_ACTIVE)
    .thenAccept(updated -> log.info("Status updated"));

connectorService.setSkillContractPermissions("contract-id", List.of("camera.control"))
    .thenAccept(updated -> log.info("Permissions updated"));
```

In practice, most adapters call `observeSkillContract` once per supported command at startup, deriving the list from whichever `EdgeAdapterService` methods they've overridden (see [Edge Adapter](edge-sdk-adapter.md#capability-reporting)).

---

## Error Handling

All `ConnectorServiceImpl` methods follow a consistent error handling pattern:

1. The gRPC response includes a `hasErrors` flag.
2. If `hasErrors` is `true`, the method logs the error and returns `null` (for entity methods) or `false` (for delete methods).
3. If the gRPC call itself fails (network error, timeout), the `CompletableFuture` completes exceptionally.

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

```properties
quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010
quarkus.grpc.clients.connector-service.keep-alive-without-calls=true
```

See the [Configuration Guide](edge-sdk-configuration.md) for the complete reference.

---

## Usage Examples

### Startup Registration Pattern

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
| `upsertAssetPayload(assetSn, subAssetSn, payload)` | `CompletableFuture<AssetPayloadDTO>` | Create or update an asset payload |
| `getSchedulerById(id)` | `CompletableFuture<SchedulerDTO>` | Get scheduler by ID |
| `createScheduler(dto)` | `CompletableFuture<SchedulerDTO>` | Create a new scheduler |
| `updateScheduler(id, dto)` | `CompletableFuture<SchedulerDTO>` | Update an existing scheduler |
| `deleteScheduler(id)` | `CompletableFuture<Boolean>` | Delete a scheduler |
| `getOrganizationById(id)` | `CompletableFuture<OrganizationDTO>` | Get organization by ID |
| `observeSkillContract(contract)` | `CompletableFuture<SkillContractProtoDTO>` | Report/update a supported command's contract |
| `listSkillContracts(status, commandId)` | `CompletableFuture<List<SkillContractProtoDTO>>` | List known command contracts |
| `setSkillContractStatus(id, status)` | `CompletableFuture<SkillContractProtoDTO>` | Change a contract's lifecycle status |
| `setSkillContractPermissions(id, permissions)` | `CompletableFuture<SkillContractProtoDTO>` | Set required permissions for a contract |

Mission, task, and application management are not part of the Edge SDK's `ConnectorService` — an edge adapter *receives* task lifecycle calls (`prepareTask`/`startTask`/`stopTask`) from the platform through `EdgeAdapterService` (see [Edge Adapter](edge-sdk-adapter.md)) rather than creating or managing them itself. A customer application authors and triggers Applications/Skills through the **Client SDK** instead — see [Applications & Skills](../concepts/applications-and-skills.md).
