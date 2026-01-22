# Edge SDK Live Data
Edge SDK Live Data for Registering Broadcasters and produce Telemetry Data.

## Quickstart

### Produce Telemetry
The Produce Telemetry method already has a default implementation.

```java
import java.time.LocalDateTime;

public class TelemetryRequestData {
	private String tid;
	private String sn;
	private String assetId;
	private LocalDateTime timestamp;
	private LiveDataType type;
	private AssetTelemetryData assetTelemetry;
	private SubAssetTelemetryData subAssetTelemetry;
}

public class AssetTelemetryData {

	private String id;
	private LocalDateTime timestamp;
	private Float latitude;
	private Float longitude;
	private Float absoluteAltitude;
	private Float relativeAltitude;
	private Float environmentTemp;
	private Float insideTemp;
	private Float humidity;
	private AssetModes mode;
	private RainfallEnum rainfall;
	private SubAssetInformation subAssetInformation;
	private Boolean subAssetAtHome;
	private Boolean subAssetCharging;
	private Float subAssetPercentage;
	private Float heading;
	private Boolean debugModeOpen;
	private Boolean hasActiveManualControlSession;
	private AssetCoverStateEnum coverState;
	private Integer workingVoltage;
	private Integer workingCurrent;
	private Integer supplyVoltage;
	private Float windSpeed;
	private Boolean positionValid;
	private NetworkInformation networkInformation;
	private AirConditioner airConditioner;
	private ManualControlStateEnum manualControlState;


	public static class AirConditioner {
		private AssetAirConditionerStateEnum state;
		private Integer switchTime;
	}


	public static class NetworkInformation {
		private NetworkTypeEnum type;
		private Float rate;
		private NetworkStateQualityEnum quality;
	}

	public static class SubAssetInformation {
		private String sn;
		private String model;
		private Boolean paired;
		private Boolean online;
	}


}

public class SubAssetTelemetryData {

	private String id;
	private LocalDateTime timestamp;
	private Float latitude;
	private Float longitude;
	private Float absoluteAltitude;
	private Float relativeAltitude;
	private Float horizontalSpeed;
	private Float verticalSpeed;
	private Float windSpeed;
	private String windDirection;
	private Float heading;
	private Integer gear;
	private PayloadTelemetry payloadTelemetry;
	private BatteryInformation batteryInformation;
	private Integer heightLimit;
	private Float homeDistance;
	private Double totalMovementDistance;
	private Double totalMovementTime;
	private SubAssetMode mode;
	private String country;

	public static class BatteryInformation {
		private String percentage;
		private Integer remainingTime;
		private String returnToHomePower;
	}

	public static class PayloadTelemetry {
		private String id;
		private LocalDateTime timestamp;
		private CameraData cameraData;
		private RangeFinderData rangeFinderData;
		private SensorData sensorData;
		private String name;
	}

	public static class CameraData {
		private String currentLens;
		private Float gimbalPitch;
		private Float gimbalYaw;
		private Float zoomFactor;
	}

	public static class RangeFinderData {
		private Float targetLatitude;
		private Float targetLongitude;
		private Float targetDistance;
		private Float targetAltitude;
	}

	public static class SensorData {
		private Float targetTemperature;
	}
}

enum LiveDataType {
	ASSET_TELEMETRY,
	SUBASSET_TELEMETRY
}
```


### 1. Map data to destination POJO
Map from your schema to the desired method reference:

Destination is :

```java
import com.zequent.framework.edge.sdk.models.TelemetryRequestData;

TelemetryRequestData telemetryRequestData = new TelemetryRequestData();

```
### 2. Send it with the method here is an example:

Reactive (Mutiny)
```java

import com.zequent.framework.edge.sdk.interfaces.LiveDataService;
import com.zequent.framework.edge.sdk.models.TelemetryRequestData;
import lombok.extern.slf4j.Slf4j;

private final LiveDataService liveDataService;

@Slf4j
public void produceTelemetry() {
	TelemetryRequestData data = new TelemetryRequestData();
	liveDataService.produceTelemetryData(data)
			.subscribe().with(_ -> log.info("Success"),
                    err -> log.error("Error: ", err));
}
```
Future-based (Plain Java / CompletionStage)

```java
import com.zequent.framework.edge.sdk.interfaces.LiveDataService;
import com.zequent.framework.edge.sdk.models.TelemetryRequestData;
import lombok.extern.slf4j.Slf4j;

private final LiveDataService liveDataService;

@Slf4j
public void produceTelemetry() {
	TelemetryRequestData data = new TelemetryRequestData();
	liveDataService.produceTelemetryData(data)
            .subscribeAsCompletionStage().thenApply(data -> log.info("Success"))
            .exceptionally(err -> log.error("Error: ", err));
}
```