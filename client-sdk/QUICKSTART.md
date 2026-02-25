# Zequent Client SDK - Quick Start Guide

## For Customers: Using the SDK in Your Project

This guide shows you how to use the Zequent Client SDK in your Java/Quarkus application.

## Step 1: Add Dependency

Add the Zequent Client SDK to your `pom.xml`:

```xml
<dependency>
    <groupId>com.zqnt.sdk</groupId>
    <artifactId>client-sdk</artifactId>
    <version>1.0.0</version>
</dependency>
```

**That's it for dependencies!** Everything else is auto-configured.

## Step 2: Configuration

Create a `.env` file in your project root (or configure via `application.properties`):

```bash
# .env
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002

MISSION_AUTONOMY_SERVICE_HOST=localhost
MISSION_AUTONOMY_SERVICE_PORT=8004

LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003
```

Or copy one of our templates:
```bash
# Development
cp .env.dev.example .env

# Production with Kubernetes
cp .env.production.example .env
```

## Step 3: Use the Client

### Option A: With CDI Injection (Recommended)

Just inject `ZequentClient` - it's automatically configured!

```java
package com.example.myapp;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.remotecontrol.domains.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import java.util.concurrent.CompletableFuture;

@Path("/drone")
public class DroneController {

    @Inject
    ZequentClient client;  // ← Auto-configured from .env!

    @POST
    @Path("/takeoff")
    public CompletableFuture<TakeoffResponse> takeoff() {
        TakeoffRequest request = TakeoffRequest.builder()
            .sn("YOUR_DEVICE_SN")
            .latitude(47.3769f)
            .longitude(8.5417f)
            .altitude(100.0f)
            .build();

        return client.remoteControl().takeoff(request);
    }

    @POST
    @Path("/land")
    public CompletableFuture<RemoteControlResponse> land() {
        ReturnToHomeRequest request = ReturnToHomeRequest.builder()
            .sn("YOUR_DEVICE_SN")
            .build();

        return client.remoteControl().returnToHome(request);
    }
}
```

### Option B: Standalone (without CDI)

If you're not using Quarkus/CDI:

```java
package com.example.myapp;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.remotecontrol.domains.*;

public class MyApp {

    public static void main(String[] args) {
        try (ZequentClient client = ZequentClient.builder()
                .remoteControl()
                    .host("localhost")
                    .port(8002)
                    .done()
                .build()) {

            var request = TakeoffRequest.builder()
                .sn("YOUR_DEVICE_SN")
                .latitude(47.3769f)
                .longitude(8.5417f)
                .altitude(100.0f)
                .build();

            var response = client.remoteControl().takeoff(request).join();
            System.out.println("Success: " + response.isSuccess());
        }
    }
}
```

## Complete Example

Here's a complete REST API using the Zequent SDK:

```java
package com.example.drone;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.remotecontrol.domains.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.util.concurrent.CompletableFuture;

@Path("/api/drone")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class DroneAPI {

    @Inject
    ZequentClient zequent;  // ← Just inject!

    // Flight Operations
    @POST
    @Path("/{sn}/takeoff")
    public CompletableFuture<TakeoffResponse> takeoff(
            @PathParam("sn") String sn,
            @QueryParam("lat") float latitude,
            @QueryParam("lon") float longitude,
            @QueryParam("alt") float altitude) {

        var request = TakeoffRequest.builder()
            .sn(sn)
            .latitude(latitude)
            .longitude(longitude)
            .altitude(altitude)
            .build();

        return zequent.remoteControl().takeoff(request);
    }

    @POST
    @Path("/{sn}/goto")
    public CompletableFuture<RemoteControlResponse> goTo(
            @PathParam("sn") String sn,
            @QueryParam("lat") float latitude,
            @QueryParam("lon") float longitude,
            @QueryParam("alt") float altitude) {

        var request = GoToRequest.builder()
            .sn(sn)
            .latitude(latitude)
            .longitude(longitude)
            .altitude(altitude)
            .build();

        return zequent.remoteControl().goTo(request);
    }

    @POST
    @Path("/{sn}/return-home")
    public CompletableFuture<RemoteControlResponse> returnToHome(@PathParam("sn") String sn) {
        var request = ReturnToHomeRequest.builder()
            .sn(sn)
            .build();

        return zequent.remoteControl().returnToHome(request);
    }

    // Dock Operations
    @POST
    @Path("/{sn}/dock/open-cover")
    public CompletableFuture<RemoteControlResponse> openCover(@PathParam("sn") String sn) {
        var request = DockOperationRequest.builder()
            .sn(sn)
            .build();

        return zequent.remoteControl().openCover(request);
    }

    @POST
    @Path("/{sn}/dock/start-charging")
    public CompletableFuture<RemoteControlResponse> startCharging(@PathParam("sn") String sn) {
        var request = DockOperationRequest.builder()
            .sn(sn)
            .build();

        return zequent.remoteControl().startCharging(request);
    }
}
```

## Environment-Specific Configuration

### Development

```bash
# .env
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002
```

```bash
mvn quarkus:dev
```

### Staging (Docker Compose)

```bash
# .env
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=8002
```

```bash
docker-compose up
```

### Production (Kubernetes)

```yaml
# deployment.yaml
env:
  - name: REMOTE_CONTROL_SERVICE_USE_STORK
    value: "true"
  - name: REMOTE_CONTROL_SERVICE_STORK_NAME
    value: "remote-control-service"
  - name: REMOTE_CONTROL_SERVICE_USE_PLAINTEXT
    value: "false"
```

## Available Services

### Remote Control
```java
client.remoteControl().takeoff(...)
client.remoteControl().goTo(...)
client.remoteControl().returnToHome(...)
client.remoteControl().lookAt(...)
client.remoteControl().openCover(...)
client.remoteControl().closeCover(...)
client.remoteControl().startCharging(...)
client.remoteControl().stopCharging(...)
client.remoteControl().rebootAsset(...)
```

### Mission Autonomy
```java
client.missionAutonomy().createMission(...)
client.missionAutonomy().createTask(...)
client.missionAutonomy().startTask(...)
client.missionAutonomy().createScheduler(...)
```

### Live Data
```java
client.liveData().streamTelemetryData(request, onData, onError)
```

## Built-in Features

 **Automatic Retry** - Retries failed requests (configurable)
 **Circuit Breaker** - Prevents cascading failures
 **Load Balancing** - Round-robin, random, least-requests
 **Service Discovery** - Stork integration for Kubernetes
 **Connection Management** - Keep-alive, reconnection
 **Environment Variables** - No code changes between environments

## Configuration Reference

All settings can be configured via environment variables:

```bash
# Service Endpoints
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002
MISSION_AUTONOMY_SERVICE_HOST=localhost
MISSION_AUTONOMY_SERVICE_PORT=8004
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003

# Resilience
ZEQUENT_MAX_RETRY_ATTEMPTS=3
ZEQUENT_RETRY_DELAY_MS=1000
ZEQUENT_CIRCUIT_BREAKER_THRESHOLD=5

# Stork (for Kubernetes)
REMOTE_CONTROL_SERVICE_USE_STORK=true
REMOTE_CONTROL_SERVICE_STORK_NAME=remote-control-service

# Load Balancing
REMOTE_CONTROL_SERVICE_LOAD_BALANCER=ROUND_ROBIN  # or LEAST_REQUESTS, RANDOM
```

See [CONFIGURATION.md](CONFIGURATION.md) for complete reference.

## Troubleshooting

### ZequentClient not injecting?

**Check 1:** Make sure you have Quarkus Arc (CDI) in your pom.xml:
```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-arc</artifactId>
</dependency>
```

**Check 2:** Verify your class has a CDI scope:
```java
@ApplicationScoped  // or @RequestScoped, @Singleton
public class MyService {
    @Inject
    ZequentClient client;
}
```

### Connection refused?

**Check:** Services are running on configured ports:
```bash
echo $REMOTE_CONTROL_SERVICE_HOST
echo $REMOTE_CONTROL_SERVICE_PORT
telnet $REMOTE_CONTROL_SERVICE_HOST $REMOTE_CONTROL_SERVICE_PORT
```

### Configuration not loading?

**Check:** .env file is in project root and properly formatted:
```bash
ls -la .env
cat .env
```

## Support

For issues or questions:
- Documentation: [CONFIGURATION.md](CONFIGURATION.md)
- GitHub Issues: https://github.com/Zequent/zequent-framework/issues
- Email: support@zequent.com

## Summary

1. **Add dependency** to `pom.xml`
2. **Create `.env`** with service endpoints
3. **Inject `ZequentClient`** in your code
4. **Use it!** `client.remoteControl().takeoff(...)`

That's all! No interfaces to implement, no complex setup! 
