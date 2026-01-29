# Minimal Customer Example

## Schritt für Schritt: Dein erstes Projekt mit Zequent Client SDK

### 1. Neues Quarkus Projekt erstellen

```bash
mvn io.quarkus:quarkus-maven-plugin:3.17.4:create \
    -DprojectGroupId=com.example \
    -DprojectArtifactId=drone-app \
    -Dextensions="resteasy-reactive-jackson,arc"

cd drone-app
```

### 2. Zequent Client SDK Dependency hinzufügen

Öffne `pom.xml` und füge hinzu:

```xml
<dependency>
    <groupId>com.zequent.framework.client.sdk</groupId>
    <artifactId>java-client-sdk</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### 3. .env Datei erstellen

```bash
# .env im Projekt-Root
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=9091
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=9093
```

### 4. Deine erste API schreiben

Erstelle `src/main/java/com/example/DroneResource.java`:

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
    ZequentClient zequent;  // ← Automatisch konfiguriert!

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

        // Automatisches Retry, Circuit Breaker, Load Balancing!
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

### 5. Starten!

```bash
mvn quarkus:dev
```

### 6. Testen

```bash
# Takeoff
curl -X POST "http://localhost:8080/drone/takeoff?sn=1581F5FKD2389A00BS8E&lat=47.3769&lon=8.5417&alt=100"

# Land
curl -X POST "http://localhost:8080/drone/land?sn=1581F5FKD2389A00BS8E"
```

## Das war's! 🎉

**Keine Interfaces implementieren!**
**Keine komplexe Konfiguration!**
**Einfach Dependency hinzufügen und loslegen!**

## Environment wechseln

### Development → Staging

```bash
# Alte .env
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=9091

# Neue .env (Docker Compose)
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=9091
```

**Kein Code-Change!** Einfach neu starten:
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

**Immer noch kein Code-Change!** Nur Deployment Config.

## Vollständige Projektstruktur

```
drone-app/
├── pom.xml                      # Mit Zequent SDK dependency
├── .env                         # Service Configuration
├── src/
│   └── main/
│       ├── java/
│       │   └── com/example/
│       │       └── DroneResource.java   # Deine API
│       └── resources/
│           └── application.properties   # Optional: Defaults
└── docker-compose.yml           # Optional: Für Staging
```

## Weitere Beispiele

### Service mit Business Logic

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

### WebSocket für Live Telemetry

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

## Das wichtigste:

### ✅ Was der Kunde bekommt:

1. **Dependency hinzufügen** → Fertig!
2. **`@Inject ZequentClient`** → Auto-konfiguriert!
3. **Environment via `.env`** wechseln → Kein Code-Change!
4. **Alle Features inklusive:**
   - Retry Logic
   - Circuit Breaker
   - Load Balancing
   - Service Discovery
   - Connection Management

### ❌ Was der Kunde NICHT machen muss:

1. ~~Interfaces implementieren~~
2. ~~Channels manuell erstellen~~
3. ~~gRPC Stubs konfigurieren~~
4. ~~Retry Logic schreiben~~
5. ~~Circuit Breaker implementieren~~
6. ~~Code für Environment-Switches ändern~~

## Support

- 📖 Komplette Docs: [CONFIGURATION.md](CONFIGURATION.md)
- 🚀 Quick Start: [QUICKSTART.md](../../../docs/QUICKSTART.md)
- 💬 GitHub: https://github.com/Zequent/zequent-framework
