# Zequent Client SDK (Go) — Connector

`connector.New(conn)` gives your Go application direct access to the platform's system of record:
asset lookup, the Skill Registry, scheduler management, technical configuration, and operational
policies. See [CONNECTOR.md](CONNECTOR.md) / [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md) for the
Java/Python equivalents — same backend service, same underlying data.

Every method takes a `context.Context` and returns `(*Result, error)`; there's no separate
`hasErrors` flag to check the way Java/Python expose — `err` covers both transport failures and
platform-side business errors (see "Error handling" below).

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "github.com/Zequent/zqnt-client-sdk-go/connector"
)

conn, _ := grpc.NewClient("localhost:8010", grpc.WithTransportCredentials(insecure.NewCredentials()))
c := connector.New(conn)
```

## Assets

| Method | Purpose |
| --- | --- |
| `GetAssetBySn(ctx, sn)` | Look up an asset by its serial number |

```go
asset, err := c.GetAssetBySn(ctx, "YOUR_DEVICE_SN")
```

This package only exposes read access to assets so far — registration/update is normally done by an
edge adapter, not a customer app (see the edge SDK docs).

## Skill Contracts

Every connected asset self-reports which commands it actually supports through its edge adapter —
that's a **Skill Contract**. Customer applications typically only need to *read* this registry (e.g.
to build a UI that only shows buttons for commands an asset actually supports); an edge adapter is
what *writes* to it.

| Method | Purpose |
| --- | --- |
| `ObserveSkillContract(ctx, contract)` | Register/update a command contract (normally called by an edge adapter) |
| `ListSkillContracts(ctx, status, commandID)` | List known command contracts, optionally filtered by status and/or command ID |
| `SetSkillContractStatus(ctx, id, status)` | Change a contract's status (e.g. deprecate it) |
| `SetSkillContractPermissions(ctx, id, requiredPermissions)` | Set which roles/permissions are required to invoke a command |

```go
contracts, err := c.ListSkillContracts(ctx, nil, "flight.takeoff")
for _, contract := range contracts {
    fmt.Println(contract.GetCommandId(), contract.GetSchemaVersion())
}
```

Pass `nil` for `status` to list every status; pass `""` for `commandID` to list every command.

## Schedulers

Schedulers define when and how often a Skill or command runs (see
[Applications & Skills](../concepts/applications-and-skills.md)).

| Method | Purpose |
| --- | --- |
| `GetScheduler(ctx, schedulerID)` | Get a scheduler by ID |
| `ListSchedulers(ctx)` | List every scheduler |
| `CreateScheduler(ctx, scheduler)` | Create one scheduler |
| `CreateSchedulers(ctx, schedulers)` | Create several in one call |
| `UpdateScheduler(ctx, schedulerID, scheduler)` | Update a scheduler |
| `DeleteScheduler(ctx, schedulerID)` | Delete one scheduler |
| `DeleteSchedulers(ctx, schedulerIDs)` | Delete several in one call |

```go
scheduler := &schedulerdto.SchedulerProtoDTO{
    Name: "Nightly patrol",
    // Cron, AssetSn, etc. — see SchedulerProtoDTO for the full field set
}
created, err := c.CreateScheduler(ctx, scheduler)
```

## Technical configuration & policies

Read-only lookups useful when your application needs to mirror platform-side configuration or
operational policy (e.g. no-fly zones, altitude limits) rather than hard-coding it.

| Method | Purpose |
| --- | --- |
| `GetActivePoliciesByType(ctx, policyType)` | Fetch active operational policies of a given type |
| `GetAllActivePolicies(ctx)` | Fetch every active operational policy |
| `GetTechnicalConfigs(ctx, scope, scopeTarget)` | Fetch technical configuration values for a scope |

## Error handling

Unlike the Java/Python SDKs (which surface a `hasErrors`/`error` pair on the response and leave
checking it to you), this Go package unwraps that convention internally — a non-nil `error` return
already carries the platform-side message:

```go
asset, err := c.GetAssetBySn(ctx, sn)
if err != nil {
    // err's message already includes the wire-level error text, e.g.
    // "connector: GetAssetBySn: asset not found: YOUR_DEVICE_SN"
    log.Printf("asset lookup failed: %v", err)
    return
}
```
