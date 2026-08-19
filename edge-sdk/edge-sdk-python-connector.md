# Edge SDK (Python) — Connector

`ConnectorClient` gives an edge adapter access to the platform's asset registry and Skill Contract registry over gRPC — what an adapter itself needs (registering its own asset, watching asset state, reporting supported commands), not general mission/task management. Mission/task CRUD was retired from `ConnectorService` in favor of the Applications/Skills execution model — see [Applications & Skills](../concepts/applications-and-skills.md), used from the **Client SDK** by customer applications.

For Java, see [edge-sdk-connector.md](edge-sdk-connector.md).

---

## Lifecycle

```python
from edge_sdk import ConnectorClient

conn = ConnectorClient(host="localhost", port=8010)
await conn.connect()
try:
    asset = await conn.get_asset_by_sn("DOCK-1")
finally:
    await conn.close()
```

---

## Asset registration

```python
from edge_sdk import Asset, AssetType, AssetVendor

asset = Asset(
    sn="DOCK-1",
    name="Roof Dock 1",
    type=AssetType.DOCK,
    vendor=AssetVendor.DJI,
)

asset_id = await conn.register_asset(asset)
if asset_id is None:
    log.error("Asset registration failed")
```

## Asset lookup

```python
asset = await conn.get_asset_by_sn("DOCK-1")
if asset is None:
    log.warning("Asset not found")
```

## Watching asset state

Subscribe to the platform's asset-monitoring stream — useful if your adapter process needs to react to changes made elsewhere (e.g. through the Admin Console):

```python
async for assets in conn.watch_assets():
    for asset in assets:
        log.info("Asset update: %s -> %s", asset.sn, asset.status)
```

The stream runs until cancelled or the server closes it; wrap it in your own retry loop if you want automatic reconnection.

---

## Skill Contracts

A Skill Contract is your adapter's self-reported declaration of which commands it supports, with their parameter schema. Reporting this accurately is what lets the Admin Console warn an operator at Skill-authoring time — "this asset doesn't support ChangeLens" — instead of failing at execution time.

```python
await conn.observe_skill_contract(contract)          # upsert a supported command's contract
contracts = await conn.list_skill_contracts(status="ACTIVE")
await conn.set_skill_contract_status(contract_id, "ACTIVE")
await conn.set_skill_contract_permissions(contract_id, ["camera.control"])
```

These methods work with the raw generated `SkillContractProtoDTO` type rather than a plain-Python dataclass — the contract shape (input/output schema, errors, events, requirements, source) is already fully typed by the protobuf definition.

---

## Error handling

`get_asset_by_sn` and `register_asset` return `None` on a business-level failure (not found / registration rejected) rather than raising — check for `None` explicitly. Transport-level failures (`UNAVAILABLE`, timeouts) raise `grpc.aio.AioRpcError`:

```python
import grpc

try:
    asset = await conn.get_asset_by_sn("DOCK-99")
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.UNAVAILABLE:
        log.warning("Connector service unreachable; retrying")
    raise
else:
    if asset is None:
        asset_id = await conn.register_asset(default_asset)
```

---

## Typical startup pattern

```python
async def boot(adapter):
    conn = ConnectorClient(host=os.environ["CONNECTOR_SERVICE_HOST"], port=int(os.environ.get("CONNECTOR_SERVICE_PORT", 8010)))
    await conn.connect()
    asset = await conn.get_asset_by_sn(os.environ["ZEQUENT_EDGE_SN"])
    if asset is None:
        asset_id = await conn.register_asset(default_asset_from_env())
    adapter.bind_connector(conn)
```
