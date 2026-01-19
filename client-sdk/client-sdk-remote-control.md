# Client SDK Remote Control
Client SDK Remote Control for executing commands and controlling assets remotely.

## Quickstart

### Remote Control Operations
The Remote Control service provides methods for operations, manual control and asset management.

```java
public class RemoteControlResponse {
	private boolean success;
	private String message;
	private String tid;
	private String sn;
	private String assetId;
	private ErrorInfo error;
	private ProgressInfo progress;

	public static class ErrorInfo {
		private String errorCode;
		private String errorMessage;
		private int timestamp;
	}

	public static class ProgressInfo {
		private float progress;
		private String state;
		private float leftTimeInSeconds;
	}
}

public class TakeoffResponse {
	private boolean success;
	private String message;
	private String tid;
	private String sn;
	private String assetId;
	private ErrorInfo error;
	private ProgressInfo progress;
}
```

### 1. Inject RemoteControl service

```java
import com.zequent.framework.client.sdk.remotecontrol.RemoteControl;

@Inject
RemoteControl remoteControl;
```

## Flight Operations

### Takeoff

```java
import com.zequent.framework.client.sdk.models.TakeoffRequest;

public void takeoff(String sn, String assetId) {
	var request = TakeoffRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.latitude(40.7222f)
		.longitude(-74.0040f)
		.altitude(50.0f)
		.missionId("M-ID1")
		.taskId("T-ID1")
		.build();
	
	var response = remoteControl.takeoff(request);
	if (response.isSuccess()) {
		log.info("Takeoff initiated successfully");
	}
}
```

### Go To Location

```java
import com.zequent.framework.client.sdk.models.GoToRequest;

public void goToLocation(String sn, String assetId) {
	var request = GoToRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.latitude(40.7189f)
		.longitude(-53.9551f)
		.altitude(40.0f)
		.missionId("M-ID1")
		.taskId("T-ID1")
		.build();
	
	var response = remoteControl.goTo(request);
	if (response.isSuccess()) {
		log.info("Navigation started");
	}
}
```

### Return To Home

```java
import com.zequent.framework.client.sdk.models.ReturnToHomeRequest;

public void returnHome(String sn, String assetId) {
	var request = ReturnToHomeRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.altitude(60.0f)
		.missionId("M-ID2")
		.taskId("T-ID2")
		.build();
	
	var response = remoteControl.returnToHome(request);
	if (response.isSuccess()) {
		log.info("Returning to home");
	}
}
```

### Look At Target

```java
import com.zequent.framework.client.sdk.models.LookAtRequest;

public void lookAtTarget(String sn, String assetId) {
	var request = LookAtRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.latitude(40.7580f)
		.longitude(-73.9855f)
		.altitude(0.0f)
		.build();
	
	var response = remoteControl.lookAt(request);
	if (response.isSuccess()) {
		log.info("Camera pointed at target");
	}
}
```

## Manual Control

### Enter Manual Control

```java
import com.zequent.framework.client.sdk.models.ManualControlRequest;

public void enterManualControl(String sn, String assetId) {
	var request = ManualControlRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.clientId("C-ID1")
		.userId("U-ID6")
		.sessionId("S-ID7")
		.reason("Manual inspection")
		.build();
	
	var response = remoteControl.enterManualControl(request);
	if (response.isSuccess()) {
		log.info("Manual control activated");
	}
}
```

### Exit Manual Control

```java
public void exitManualControl(String sn, String assetId) {
	var request = ManualControlRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.clientId("C-ID2")
		.userId("U-ID3")
		.sessionId("S-ID1")
		.build();
	
	var response = remoteControl.exitManualControl(request);
	if (response.isSuccess()) {
		log.info("Manual control deactivated");
	}
}
```

## Dock Operations

### Open Cover

```java
import com.zequent.framework.client.sdk.models.DockOperationRequest;

public void openCover(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.force(false)
		.build();
	
	var response = remoteControl.openCover(request);
	if (response.isSuccess()) {
		log.info("Cover opened");
	}
}
```

### Close Cover

```java
public void closeCover(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.force(false)
		.build();
	
	var response = remoteControl.closeCover(request);
	if (response.isSuccess()) {
		log.info("Cover closed");
	}
}
```

### Start Charging

```java
public void startCharging(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.build();
	
	var response = remoteControl.startCharging(request);
	if (response.isSuccess()) {
		log.info("Charging started");
	}
}
```

### Stop Charging

```java
public void stopCharging(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.build();
	
	var response = remoteControl.stopCharging(request);
	if (response.isSuccess()) {
		log.info("Charging stopped");
	}
}
```

## Asset Operations

### Reboot Asset

```java
public void rebootAsset(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.force(false)
		.build();
	
	var response = remoteControl.rebootAsset(request);
	if (response.isSuccess()) {
		log.info("Asset reboot initiated");
	}
}
```

### Boot SubAsset

```java
public void bootSubAsset(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.build();
	
	var response = remoteControl.bootSubAsset(request);
	if (response.isSuccess()) {
		log.info("SubAsset boot initiated");
	}
}
```

### Debug Mode

```java
public void enableDebugMode(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.force(true)
		.build();
	
	var response = remoteControl.debugMode(request);
	if (response.isSuccess()) {
		log.info("Debug mode enabled");
	}
}
```

### Change Air Conditioner Mode

```java
public void changeAcMode(String sn, String assetId) {
	var request = DockOperationRequest.builder()
		.sn(sn)
		.assetId(assetId)
		.build();
	
	var response = remoteControl.changeAcMode(request);
	if (response.isSuccess()) {
		log.info("AC mode changed");
	}
}
```
