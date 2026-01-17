# Client SDK Live Data
Client SDK Live Data for consuming real-time telemetry streams.

## Quickstart

### Stream Telemetry
The Stream Telemetry method takes live telemetry data from devices.

```java
public class LiveDataTelemetryResponse {
	private String tid;
	private String timestamp;
	private boolean hasErrors;
	private String sn;
	private String assetId;
	private AssetTelemetry assetTelemetry;
	private SubAssetTelemetry subAssetTelemetry;
	private GlobalErrorMessage error;
}

public class AssetTelemetry {
	private String id;
	private String timestamp;
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

public class SubAssetTelemetry {
	private String id;
	private String timestamp;
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
		private String timestamp;
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

enum LiveDataServiceCommand {
	START_TELEMETRY_STREAM,
	STOP_TELEMETRY_STREAM
}
```

### 1. Inject LiveData service

```java
import com.zequent.framework.client.sdk.livedata.LiveData;

@Inject
LiveData liveData;
```

### 2. Start streaming with request

```java
import com.zequent.framework.common.proto.LiveDataServiceCommand;
import com.zequent.framework.common.proto.RequestBase;
import com.zequent.framework.services.livedata.proto.LiveDataStreamTelemetryRequest;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public void startStream(String sn) {
	var request = LiveDataStreamTelemetryRequest.newBuilder()
		.setBase(RequestBase.newBuilder()
			.setSn(sn)
			.setTid(UUID.randomUUID().toString())
			.setTimestamp((int) System.currentTimeMillis())
			.build())
		.setFrequencyMs(100)
		.setDuration(0)
		.setCommand(LiveDataServiceCommand.START_TELEMETRY_STREAM)
		.build();
	
	liveData.streamTelemetryData(request, response -> {
		if (response.hasSubAssetTelemetry()) {
			var telemetry = response.getSubAssetTelemetry();
			log.info("SubAsset - Alt: {}m, Battery: {}%", 
				telemetry.getAbsoluteAltitude(),
				telemetry.getBatteryInformation().getPercentage());
		}
		
		if (response.hasAssetTelemetry()) {
			var telemetry = response.getAssetTelemetry();
			log.info("Asset - Pos: {},{} Temp: {}°C", 
				telemetry.getLatitude(),
				telemetry.getLongitude(),
				telemetry.getEnvironmentTemp());
		}
	});
}
```

### 3. Stop streaming

```java
public void stopStream(String sn) {
	var request = LiveDataStreamTelemetryRequest.newBuilder()
		.setBase(RequestBase.newBuilder()
			.setSn(sn)
			.setTid(UUID.randomUUID().toString())
			.setTimestamp((int) System.currentTimeMillis())
			.build())
		.setCommand(LiveDataServiceCommand.STOP_TELEMETRY_STREAM)
		.build();
	
	liveData.streamTelemetryData(request, r -> log.info("Stream stopped"));
}
```
