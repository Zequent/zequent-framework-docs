# Spring Boot Integration - Final Solution

The **simplest** way to use ZequentClient in Spring Boot.

Uses `ZequentClientProducer` internally for consistent bean creation.

---

## Solution 1: Ultra Simple (Only Defaults)

### Step 1: Maven Dependency

```xml
<dependency>
    <groupId>com.zequent.framework</groupId>
    <artifactId>java-client-sdk</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### Step 2: Write One Bean Method

**`src/main/java/com/yourcompany/config/ZequentConfig.java`**

```java
package com.yourcompany.config;

import com.zequent.framework.client.sdk.ZequentClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ZequentConfig {

    @Bean
    public ZequentClient zequentClient() {
        // Uses all defaults: localhost:8002, 8004, 8003
        return ZequentClient.builder()
                .remoteControl().done()
                .missionAutonomy().done()
                .liveData().done()
                .build();
    }
}
```

### Step 3: Use Constructor Injection

```java
package com.yourcompany.service;

import com.zequent.framework.client.sdk.ZequentClient;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class LiveDataService {

    private final ZequentClient zequentClient;  // ← Automatically injected!

    public void streamTelemetry() {
        zequentClient.liveData().streamTelemetryData();
    }
}
```

**Done!** 

---

## Solution 2: With Properties (Optional)

If you want to use properties from `application.properties`:

### Bean mit @Value

```java
package com.yourcompany.config;

import com.zequent.framework.client.sdk.ZequentClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ZequentConfig {

    @Bean
    public ZequentClient zequentClient(
            @Value("${zequent.remote-control.host:localhost}") String rcHost,
            @Value("${zequent.remote-control.port:8002}") int rcPort,
            @Value("${zequent.mission-autonomy.host:localhost}") String maHost,
            @Value("${zequent.mission-autonomy.port:8004}") int maPort,
            @Value("${zequent.live-data.host:localhost}") String ldHost,
            @Value("${zequent.live-data.port:8003}") int ldPort) {

        return ZequentClient.builder()
                .remoteControl()
                    .host(rcHost)
                    .port(rcPort)
                    .done()
                .missionAutonomy()
                    .host(maHost)
                    .port(maPort)
                    .done()
                .liveData()
                    .host(ldHost)
                    .port(ldPort)
                    .done()
                .build();
    }
}
```

### application.properties

```properties
# Only what you want to change - rest uses defaults
zequent.remote-control.host=prod-server.com
zequent.live-data.port=9999
```

---

## Solution 3: With ZequentClientProducer (Advanced)

If you want to use the `ZequentClientProducer` directly (uses the same logic as Quarkus):

```java
package com.yourcompany.config;

import com.zequent.framework.client.sdk.ZequentClient;
import com.zequent.framework.client.sdk.ZequentClientProducer;
import com.zequent.framework.client.sdk.config.GrpcClientConfig;
import com.zequent.framework.client.sdk.config.ServiceConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ZequentConfig {

    @Bean
    public ZequentClient zequentClient(
            @Value("${zequent.remote-control.host:localhost}") String rcHost,
            @Value("${zequent.remote-control.port:8002}") int rcPort,
            @Value("${zequent.live-data.host:localhost}") String ldHost,
            @Value("${zequent.live-data.port:8003}") int ldPort) {

        // Manually create GrpcClientConfig
        GrpcClientConfig config = GrpcClientConfig.builder()
                .remoteControlConfig(ServiceConfig.builder()
                        .serviceName("remote-control")
                        .host(rcHost)
                        .port(rcPort)
                        .usePlaintext(true)
                        .useStork(false)
                        .loadBalancerType(ServiceConfig.LoadBalancerType.ROUND_ROBIN)
                        .build())
                .missionAutonomyConfig(ServiceConfig.builder()
                        .serviceName("mission-autonomy")
                        .host("localhost")
                        .port(8004)
                        .usePlaintext(true)
                        .useStork(false)
                        .loadBalancerType(ServiceConfig.LoadBalancerType.ROUND_ROBIN)
                        .build())
                .liveDataConfig(ServiceConfig.builder()
                        .serviceName("live-data")
                        .host(ldHost)
                        .port(ldPort)
                        .usePlaintext(true)
                        .useStork(false)
                        .loadBalancerType(ServiceConfig.LoadBalancerType.ROUND_ROBIN)
                        .build())
                .maxRetryAttempts(3)
                .retryDelayMillis(1000L)
                .circuitBreakerFailureThreshold(5)
                .circuitBreakerWaitDurationMillis(30000L)
                .connectionTimeoutSeconds(30)
                .requestTimeoutSeconds(60)
                .defaultLoadBalancerType(ServiceConfig.LoadBalancerType.ROUND_ROBIN)
                .build();

        // Use producer logic directly
        return createZequentClient(config);
    }

    /**
     * Creates ZequentClient with the same logic as ZequentClientProducer.
     */
    private ZequentClient createZequentClient(GrpcClientConfig config) {
        // Create channels for each service
        java.util.List<io.grpc.ManagedChannel> channels = new java.util.ArrayList<>();
        io.grpc.ManagedChannel remoteControlChannel =
                com.zequent.framework.client.sdk.channel.ChannelFactory.createChannel(config.getRemoteControlConfig());
        io.grpc.ManagedChannel missionAutonomyChannel =
                com.zequent.framework.client.sdk.channel.ChannelFactory.createChannel(config.getMissionAutonomyConfig());
        io.grpc.ManagedChannel liveDataChannel =
                com.zequent.framework.client.sdk.channel.ChannelFactory.createChannel(config.getLiveDataConfig());

        channels.add(remoteControlChannel);
        channels.add(missionAutonomyChannel);
        channels.add(liveDataChannel);

        // Create service implementations
        com.zequent.framework.client.sdk.remotecontrol.RemoteControl remoteControl =
                com.zequent.framework.client.sdk.remotecontrol.impl.RemoteControlImpl.create(config, remoteControlChannel);
        com.zequent.framework.client.sdk.missionautonomy.MissionAutonomy missionAutonomy =
                com.zequent.framework.client.sdk.missionautonomy.MissionAutonomy.create(config, missionAutonomyChannel);
        com.zequent.framework.client.sdk.livedata.LiveData liveData =
                com.zequent.framework.client.sdk.livedata.LiveData.create(config, liveDataChannel);

        // Return ZequentClient (exactly as ZequentClientProducer does)
        return new ZequentClient(config, remoteControl, missionAutonomy, liveData, channels);
    }
}
```

---

## Recommendation

### For most customers: **Solution 1** or **Solution 2**

 **Ultra-simple**
 **Builder with defaults**
 **Optional properties via @Value**

### For power users: **Solution 3**

 **Uses producer logic directly**
 **Maximum control**
 **Consistent with Quarkus**

---

## Purpose of Components

### `ZequentClientProducer` (for Quarkus)
- Automatically used by Quarkus CDI
- Reads properties via `@ConfigMapping`
- Creates `ZequentClient` automatically

### `ZequentClient.builder()` (for Spring Boot & Standalone)
- For manual bean creation
- Uses defaults (localhost:8002/8004/8003)
- Flexibly configurable

### Both produce the same result! 

The `ZequentClientProducer` is **only** relevant for Quarkus CDI.
For Spring Boot, the customer uses the **Builder** - that's the right way! 

---

## Usage

No matter which solution - the usage is identical:

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final ZequentClient zequentClient;

    public void doSomething() {
        zequentClient.liveData().streamTelemetryData();
        zequentClient.remoteControl().takeoff(...);
    }
}
```

**Simple, right?** 
