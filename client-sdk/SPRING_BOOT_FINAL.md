# Spring Boot Integration - Finale Lösung

Die **einfachste** Möglichkeit, ZequentClient in Spring Boot zu verwenden.

Nutzt intern `ZequentClientProducer` für konsistente Bean-Erstellung.

---

## Lösung 1: Ultra Simple (Nur Defaults)

### Schritt 1: Maven Dependency

```xml
<dependency>
    <groupId>com.zequent.framework</groupId>
    <artifactId>java-client-sdk</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

### Schritt 2: Eine Bean-Methode schreiben

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
        // Nutzt alle Defaults: localhost:8002, 8004, 8003
        return ZequentClient.builder()
                .remoteControl().done()
                .missionAutonomy().done()
                .liveData().done()
                .build();
    }
}
```

### Schritt 3: Constructor Injection verwenden

```java
package com.yourcompany.service;

import com.zequent.framework.client.sdk.ZequentClient;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class LiveDataService {

    private final ZequentClient zequentClient;  // ← Automatisch injiziert!

    public void streamTelemetry() {
        zequentClient.liveData().streamTelemetryData();
    }
}
```

**Fertig!** ✅

---

## Lösung 2: Mit Properties (Optional)

Wenn du Properties aus `application.properties` nutzen willst:

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
# Nur was du ändern willst - Rest nutzt Defaults
zequent.remote-control.host=prod-server.com
zequent.live-data.port=9999
```

---

## Lösung 3: Mit ZequentClientProducer (Fortgeschritten)

Wenn du den `ZequentClientProducer` direkt nutzen willst (nutzt dieselbe Logik wie Quarkus):

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

        // Manuell GrpcClientConfig erstellen
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

        // Nutze Producer-Logik direkt
        return createZequentClient(config);
    }

    /**
     * Erstellt ZequentClient mit derselben Logik wie ZequentClientProducer.
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

## Empfehlung

### Für die meisten Kunden: **Lösung 1** oder **Lösung 2**

✅ **Ultra-einfach**
✅ **Builder mit Defaults**
✅ **Optional Properties via @Value**

### Für Power-User: **Lösung 3**

✅ **Nutzt Producer-Logik direkt**
✅ **Maximale Kontrolle**
✅ **Konsistent mit Quarkus**

---

## Zweck der Komponenten

### `ZequentClientProducer` (für Quarkus)
- Wird automatisch von Quarkus CDI verwendet
- Liest Properties via `@ConfigMapping`
- Erstellt `ZequentClient` automatisch

### `ZequentClient.builder()` (für Spring Boot & Standalone)
- Für manuelle Bean-Erstellung
- Nutzt Defaults (localhost:8002/8004/8003)
- Flexibel konfigurierbar

### Beide produzieren dasselbe Ergebnis! ✅

Der `ZequentClientProducer` ist **nur** für Quarkus CDI relevant.
Für Spring Boot nutzt der Kunde den **Builder** - das ist der richtige Weg! 🎯

---

## Verwendung

Egal welche Lösung - die Verwendung ist identisch:

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

**Einfach, oder?** 🎉
