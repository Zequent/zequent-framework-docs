# Zequent Client SDK — Functional Responses

Every RemoteControl call returns a response as soon as the edge adapter replies — but "success" in that response is easy to misread as "the device did the thing." This page explains what a command response actually represents, and how to confirm what really happened on the device.

For Python, see [FUNCTIONAL_RESPONSES_PYTHON.md](FUNCTIONAL_RESPONSES_PYTHON.md).

## Command calls are synchronous, not fire-and-forget

`client.remoteControl().takeoff(request)` doesn't return the moment your call is accepted — it blocks (behind the `CompletableFuture`) until the device's own command handler has actually responded. There's no separate "ack" that arrives later on some other channel: the `CompletableFuture` you get back *is* that answer.

## What `success` actually means

A successful response means the adapter's handler for that command returned without error — nothing more. It's entirely up to the adapter's own implementation whether that means "the flight controller accepted the takeoff command" or "the drone has physically left the ground." The platform doesn't distinguish the two, and neither does the response:

```java
TakeoffResponse response = client.remoteControl().takeoff(request).get();

if (response.isSuccess()) {
    // The adapter accepted and initiated the command.
    // This does NOT mean the drone is airborne yet.
    System.out.println(response.getMessage()); // e.g. "Takeoff initiated"
} else {
    System.out.println(response.getError().getErrorMessage());
}
```

Treat a successful response as "the command was handed off and accepted," not as confirmation of the physical outcome. For that, you need telemetry.

**What that response actually looks like**, e.g. printed via `System.out.println(response)`:

```
TakeoffResponse(success=true, message=Takeoff initiated, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=550e8400-e29b-41d4-a716-446655440000, error=null, progress=null)
```

A rejected command looks the same shape, with `error` populated instead of `message`:

```
TakeoffResponse(success=false, message=null, tid=a1b2c3d4-1234-5678-9abc-def012345678, sn=ZQT-DOCK-0417, assetId=550e8400-e29b-41d4-a716-446655440000, error=ErrorInfo(errorCode=ERROR_CODE_ASSET, errorMessage=Asset ZQT-DOCK-0417 is not connected, timestamp=2026-08-26T18:41:52), progress=null)
```

## Confirming what actually happened, via telemetry

The response to a command call isn't the source of truth for device state — the telemetry stream is. Subscribe with `client.liveData()` and watch the relevant field, rather than assuming the command response means the job is done:

| Command(s) | Field to watch | Confirms it worked |
| --- | --- | --- |
| `takeoff()` | `SubAssetTelemetry.mode` | reaches `SUBASSET_MODE_TAKEOFF_FINISHED` (after `TAKEOFF_PREPARE` → `TAKEOFF_AUTO`) |
| `openCover()` / `closeCover()` | `AssetTelemetry.coverState` | becomes `COVER_STATE_OPENED` / `COVER_STATE_CLOSED` |
| `startCharging()` / `stopCharging()` | `AssetTelemetry.subAssetCharging` | becomes `true` / `false` |
| `enterManualControl()` / `exitManualControl()` | `AssetTelemetry.hasActiveManualControlSession` | becomes `true` / `false` |
| `returnToHome()` | `SubAssetTelemetry.mode` | moves through `RETURN_AUTO` → `LANDING_AUTO` → back to `IDLE` once docked |

Worked example, using `takeoff()`:

```java
client.remoteControl().takeoff(request)
    .thenAccept(response -> {
        if (!response.isSuccess()) {
            System.out.println("Adapter rejected takeoff: " + response.getError().getErrorMessage());
            return;
        }

        // Command was accepted — now watch telemetry for what actually happens.
        var telemetryRequest = new StreamTelemetryRequest();
        telemetryRequest.setSn(request.getSn());

        client.liveData().streamTelemetryData(telemetryRequest, telemetry -> {
            var subAsset = telemetry.getSubAssetTelemetry();
            if (subAsset != null && subAsset.getMode() == SubAssetMode.SUBASSET_MODE_TAKEOFF_FINISHED) {
                System.out.println("Drone is airborne.");
            }
        });
    });
```

**What a telemetry frame actually looks like** — one `StreamTelemetryResponse` from the drone mid-takeoff (only the populated fields are shown; `SubAssetTelemetryData` has more, e.g. `windSpeed`, `heightLimit`, `payloadTelemetry`):

```
StreamTelemetryResponse(
    tid=f47ac10b-58cc-4372-a567-0e02b2c3d479,
    timestamp=2026-08-26T18:42:07.912Z,
    hasErrors=false,
    sn=ZQT-DOCK-0417,
    assetId=550e8400-e29b-41d4-a716-446655440000,
    assetTelemetry=null,
    subAssetTelemetry=SubAssetTelemetryData(
        id=ZQT-DRONE-1123,
        latitude=41.015137,
        longitude=28.979530,
        absoluteAltitude=42.6,
        relativeAltitude=12.4,
        verticalSpeed=1.1,
        heading=187.5,
        mode=SUBASSET_MODE_TAKEOFF_AUTO,
        batteryInformation=BatteryInformation(percentage=78, remainingTime=1320, returnToHomePower=22)
    ),
    error=null
)
```

Every frame carries exactly one of `assetTelemetry` (the dock) or `subAssetTelemetry` (the drone) — never both.

See [Connector](CONNECTOR.md) for looking up asset state on demand instead of streaming it, and [Quickstart](QUICKSTART.md) for how `client.remoteControl()` / `client.liveData()` get wired up in the first place.
