# Minimal Customer Example

## Step by Step: Your First Project with Zequent Client SDK

### 1. Create New Quarkus Project

```bash
mvn io.quarkus:quarkus-maven-plugin:3.17.4:create \
    -DprojectGroupId=com.example \
    -DprojectArtifactId=drone-app \
    -Dextensions="resteasy-reactive-jackson,arc"

cd drone-app
```

### 2. Add Zequent Client SDK Dependency

Open `pom.xml` and add:

```xml
<dependency>
    <groupId>com.zequent.framework.client.sdk</groupId>
    <artifactId>java-client-sdk</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### 3. Create .env File

```bash
# .env in project root
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=9091
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=9093
```

### 4. Write Your First API

Create `src/main/java/com/example/DroneResource.java`:

```java
package com.example;

import com.zequent.framework.client.sdk.ZequentClient;
import com.zequent.framework.client.sdk.models.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;

@Path("/drone")
@Produces(MediaType.APPLICATION_JSON)
public class DroneResource {

    @Inject
    ZequentClient zequent;  // ← Automatically configured!

    @POST
    @Path("/takeoff")
    public TakeoffResponse takeoff(
            @QueryParam("sn") String sn,
            @QueryParam("lat") double lat,
            @QueryParam("lon") double lon,
            @QueryParam("alt") double alt) {

        var request = TakeoffRequest.builder()
            .sn(sn)
            .latitude(lat)
            .longitude(lon)
            .altitude(alt)
            .build();

        // Automatic Retry, Circuit Breaker, Load Balancing!
        return zequent.remoteControl().takeoff(request);
    }

    @POST
    @Path("/land")
    public RemoteControlResponse land(@QueryParam("sn") String sn) {
        var request = ReturnToHomeRequest.builder()
            .sn(sn)
            .build();

        return zequent.remoteControl().returnToHome(request);
    }
}
```

### 5. Start!

```bash
mvn quarkus:dev
```

### 6. Test

```bash
# Takeoff
curl -X POST "http://localhost:8080/drone/takeoff?sn=YOUR_DEVICE_SN&lat=47.3769&lon=8.5417&alt=100"

# Land
curl -X POST "http://localhost:8080/drone/land?sn=YOUR_DEVICE_SN"
```

## That's it! 

**No interfaces to implement!**
**No complex configuration!**
**Just add dependency and start!**

## Switch Environment

### Development → Staging

```bash
# Old .env
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=9091

# New .env (Docker Compose)
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=9091
```

**No code change!** Just restart:
```bash
docker-compose up
```

### Staging → Production (Kubernetes)

```yaml
# deployment.yaml
env:
  - name: REMOTE_CONTROL_SERVICE_USE_STORK
    value: "true"
  - name: REMOTE_CONTROL_SERVICE_STORK_NAME
    value: "remote-control-service"
```

**Still no code change!** Only deployment config.

## Complete Project Structure

```
drone-app/
 pom.xml                      # With Zequent SDK dependency
 .env                         # Service Configuration
 src/
    main/
        java/
           com/example/
               DroneResource.java   # Your API
        resources/
            application.properties   # Optional: Defaults
 docker-compose.yml           # Optional: For Staging
```

## More Examples

### Service with Business Logic

```java
package com.example;

import com.zequent.framework.client.sdk.ZequentClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class DroneFlightService {

    @Inject
    ZequentClient zequent;

    public boolean executeMission(String sn, List<Waypoint> waypoints) {
        // 1. Takeoff
        var takeoffResponse = zequent.remoteControl().takeoff(
            TakeoffRequest.builder()
                .sn(sn)
                .latitude(waypoints.get(0).getLat())
                .longitude(waypoints.get(0).getLon())
                .altitude(100.0)
                .build()
        );

        if (!takeoffResponse.isSuccess()) {
            return false;
        }

        // 2. Fly waypoints
        for (Waypoint wp : waypoints) {
            var goToResponse = zequent.remoteControl().goTo(
                GoToRequest.builder()
                    .sn(sn)
                    .latitude(wp.getLat())
                    .longitude(wp.getLon())
                    .altitude(wp.getAlt())
                    .build()
            );

            if (!goToResponse.isSuccess()) {
                return false;
            }
        }

        // 3. Return to home
        var rthResponse = zequent.remoteControl().returnToHome(
            ReturnToHomeRequest.builder()
                .sn(sn)
                .build()
        );

        return rthResponse.isSuccess();
    }
}
```

### WebSocket for Live Telemetry

```java
package com.example;

import com.zequent.framework.client.sdk.ZequentClient;
import com.zequent.framework.services.livedata.proto.*;
import jakarta.inject.Inject;
import jakarta.websocket.*;
import jakarta.websocket.server.ServerEndpoint;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@ServerEndpoint("/ws/telemetry/{sn}")
public class TelemetryWebSocket {

    @Inject
    ZequentClient zequent;

    @OnOpen
    public void onOpen(Session session, @PathParam("sn") String sn) {
        log.info("Client connected: {}", sn);

        var request = LiveDataStreamTelemetryRequest.newBuilder()
            .addSn(sn)
            .build();

        // Stream telemetry to WebSocket client
        zequent.liveData().streamTelemetryData(
            request,
            telemetry -> {
                try {
                    session.getBasicRemote().sendText(telemetry.toString());
                } catch (Exception e) {
                    log.error("Failed to send telemetry", e);
                }
            },
            error -> log.error("Telemetry stream error", error)
        );
    }
}
```

## The Most Important:

###  What the Customer Gets:

1. **Add dependency** → Done!
2. **`@Inject ZequentClient`** → Auto-configured!
3. **Switch environment via `.env`** → No code change!
4. **All features included:**
   - Retry Logic
   - Circuit Breaker
   - Load Balancing
   - Service Discovery
   - Connection Management

###  What the Customer Does NOT Have to Do:

1. ~~Implement interfaces~~
2. ~~Manually create channels~~
3. ~~Configure gRPC stubs~~
4. ~~Write retry logic~~
5. ~~Implement circuit breaker~~
6. ~~Change code for environment switches~~

## Support

-  Complete Docs: [CONFIGURATION.md](CONFIGURATION.md)
-  Quick Start: [QUICKSTART.md](../../../docs/QUICKSTART.md)
-  GitHub: https://github.com/Zequent/zequent-framework
