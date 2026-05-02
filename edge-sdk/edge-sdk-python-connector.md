# Edge SDK (Python) — Connector

`ConnectorClient` provides asset and resource management against the platform's Connector Service over gRPC.

For Java, see [edge-sdk-connector.md](edge-sdk-connector.md).

---

## Lifecycle

```python
from edge_sdk import ConnectorClient

async with ConnectorClient(host="localhost", port=8010) as conn:
    asset = await conn.get_asset_by_sn("DOCK-1")
```

`from_env()` reads `CONNECTOR_SERVICE_HOST` / `CONNECTOR_SERVICE_PORT`:

```python
async with ConnectorClient.from_env() as conn:
    ...
```

---

## Asset registration

Most adapters call this once on startup:

```python
from edge_sdk import Asset, AssetType, AssetVendor, SubAsset

asset = Asset(
    sn="DOCK-1",
    name="Roof Dock 1",
    type=AssetType.DOCK,
    vendor=AssetVendor.DJI,
    organization_id="org-uuid",
    sub_assets=[
        SubAsset(sn="DRONE-1", parent_sn="DOCK-1", name="Mavic 3"),
    ],
)

registered = await conn.register_asset(asset)
print("registered with id:", registered.id)
```

If the asset already exists by SN, `register_asset` returns the existing record (idempotent).

---

## Asset queries

| Method                                          | Purpose                              |
|-------------------------------------------------|--------------------------------------|
| `get_asset(asset_id: str) -> Asset`             | Lookup by UUID                       |
| `get_asset_by_sn(sn: str) -> Asset`             | Lookup by serial number              |
| `list_assets(*, organization_id, ...) -> list[Asset]` | Paginated list                 |
| `update_asset(asset: Asset) -> Asset`           | Persist changes                      |
| `delete_asset(asset_id: str) -> None`           | Remove the asset                     |

```python
asset = await conn.get_asset_by_sn("DOCK-1")
asset.name = "Roof Dock 1 (renamed)"
asset = await conn.update_asset(asset)
```

---

## Sub-asset operations

```python
new_sub = await conn.add_sub_asset(parent_sn="DOCK-1", sub=SubAsset(sn="DRONE-2", ...))
await conn.remove_sub_asset(sub_id=new_sub.id)
```

---

## Error handling

All `ConnectorClient` methods raise `grpc.aio.AioRpcError` on transport / business errors. Inspect `e.code()`:

| `grpc.StatusCode` | Meaning                                           |
|-------------------|---------------------------------------------------|
| `NOT_FOUND`       | The asset / sub-asset does not exist              |
| `ALREADY_EXISTS`  | Duplicate SN at registration time                 |
| `INVALID_ARGUMENT`| Malformed request (e.g. missing required fields)  |
| `UNAVAILABLE`     | Connector service unreachable; retry with backoff |

```python
import grpc
try:
    asset = await conn.get_asset_by_sn("DOCK-99")
except grpc.aio.AioRpcError as e:
    if e.code() == grpc.StatusCode.NOT_FOUND:
        asset = await conn.register_asset(default_asset)
    else:
        raise
```

---

## Typical startup pattern

```python
async def boot(adapter):
    async with ConnectorClient.from_env() as conn:
        try:
            asset = await conn.get_asset_by_sn(os.environ["ZEQUENT_EDGE_SN"])
        except grpc.aio.AioRpcError as e:
            if e.code() != grpc.StatusCode.NOT_FOUND:
                raise
            asset = await conn.register_asset(default_asset_from_env())
        adapter.bind_asset(asset)
```

This is the Python equivalent of the Java SDK's startup beans that auto-register assets via CDI.
