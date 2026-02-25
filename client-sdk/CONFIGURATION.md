# Zequent Client SDK - Configuration Guide

## Overview

The Zequent Client SDK supports flexible configuration via **application.properties** or **environment variables**, allowing you to switch between environments (dev, staging, production) **without any code changes**.

## Configuration Methods

### Priority Order (highest to lowest):
1. **Environment Variables** (e.g., `REMOTE_CONTROL_SERVICE_HOST`)
2. **System Properties** (e.g., `-Dzequent.remote-control-service.host=...`)
3. **.env file** (automatically loaded by Quarkus)
4. **application.properties** (defaults)

## Quick Start

### 1. Choose Your Environment

Copy the appropriate `.env` template:

```bash
# Development (local services)
cp .env.dev.example .env

# Staging (Docker Compose)
cp .env.staging.example .env

# Production (Kubernetes)
cp .env.production.example .env
```

### 2. Run Your Application

```bash
mvn quarkus:dev
```

**That's it!** No code changes needed.

## Configuration Reference

### Service Configuration

Each service (Remote Control, Mission Autonomy, Live Data) has the following settings:

#### Remote Control Service

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `zequent.remote-control-service.host` | `REMOTE_CONTROL_SERVICE_HOST` | `localhost` | Service hostname |
| `zequent.remote-control-service.port` | `REMOTE_CONTROL_SERVICE_PORT` | `8002` | Service port |
| `zequent.remote-control-service.use-plaintext` | `REMOTE_CONTROL_SERVICE_USE_PLAINTEXT` | `true` | Use plaintext (no TLS) |
| `zequent.remote-control-service.use-stork` | `REMOTE_CONTROL_SERVICE_USE_STORK` | `false` | Enable Stork service discovery |
| `zequent.remote-control-service.stork-service-name` | `REMOTE_CONTROL_SERVICE_STORK_NAME` | `remote-control-service` | Stork service name |
| `zequent.remote-control-service.load-balancer-type` | `REMOTE_CONTROL_SERVICE_LOAD_BALANCER` | `ROUND_ROBIN` | Load balancer: `ROUND_ROBIN`, `RANDOM`, `LEAST_REQUESTS` |

#### Mission Autonomy Service

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `zequent.mission-autonomy-service.host` | `MISSION_AUTONOMY_SERVICE_HOST` | `localhost` | Service hostname |
| `zequent.mission-autonomy-service.port` | `MISSION_AUTONOMY_SERVICE_PORT` | `8004` | Service port |
| `zequent.mission-autonomy-service.use-plaintext` | `MISSION_AUTONOMY_SERVICE_USE_PLAINTEXT` | `true` | Use plaintext |
| `zequent.mission-autonomy-service.use-stork` | `MISSION_AUTONOMY_SERVICE_USE_STORK` | `false` | Enable Stork |
| `zequent.mission-autonomy-service.stork-service-name` | `MISSION_AUTONOMY_SERVICE_STORK_NAME` | `mission-autonomy-service` | Stork service name |
| `zequent.mission-autonomy-service.load-balancer-type` | `MISSION_AUTONOMY_SERVICE_LOAD_BALANCER` | `ROUND_ROBIN` | Load balancer type |

#### Live Data Service

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `zequent.live-data-service.host` | `LIVE_DATA_SERVICE_HOST` | `localhost` | Service hostname |
| `zequent.live-data-service.port` | `LIVE_DATA_SERVICE_PORT` | `8003` | Service port |
| `zequent.live-data-service.use-plaintext` | `LIVE_DATA_SERVICE_USE_PLAINTEXT` | `true` | Use plaintext |
| `zequent.live-data-service.use-stork` | `LIVE_DATA_SERVICE_USE_STORK` | `false` | Enable Stork |
| `zequent.live-data-service.stork-service-name` | `LIVE_DATA_SERVICE_STORK_NAME` | `live-data-service` | Stork service name |
| `zequent.live-data-service.load-balancer-type` | `LIVE_DATA_SERVICE_LOAD_BALANCER` | `ROUND_ROBIN` | Load balancer type |

### Global Resilience Settings

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `zequent.resilience.max-retry-attempts` | `ZEQUENT_MAX_RETRY_ATTEMPTS` | `3` | Maximum retry attempts |
| `zequent.resilience.retry-delay-millis` | `ZEQUENT_RETRY_DELAY_MS` | `1000` | Delay between retries (ms) |
| `zequent.resilience.circuit-breaker-failure-threshold` | `ZEQUENT_CIRCUIT_BREAKER_THRESHOLD` | `5` | Failures before circuit opens |
| `zequent.resilience.circuit-breaker-wait-duration-millis` | `ZEQUENT_CIRCUIT_BREAKER_WAIT_MS` | `30000` | Wait before retry (ms) |
| `zequent.resilience.connection-timeout-seconds` | `ZEQUENT_CONNECTION_TIMEOUT_SEC` | `30` | Connection timeout |
| `zequent.resilience.request-timeout-seconds` | `ZEQUENT_REQUEST_TIMEOUT_SEC` | `60` | Request timeout |

## Environment Examples

### Development (.env.dev.example)

```bash
# Local development - all services on localhost
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002
REMOTE_CONTROL_SERVICE_USE_PLAINTEXT=true
REMOTE_CONTROL_SERVICE_USE_STORK=false
```

**Usage:**
```bash
cp .env.dev.example .env
mvn quarkus:dev
```

### Staging - Docker Compose (.env.staging.example)

```bash
# Docker Compose - use service names
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=8002
REMOTE_CONTROL_SERVICE_USE_PLAINTEXT=true
REMOTE_CONTROL_SERVICE_USE_STORK=false
```

**docker-compose.yml:**
```yaml
services:
  remote-control-service:
    image: zequent/remote-control:latest
    ports:
      - "8002:8002"

  my-app:
    image: my-app:latest
    env_file: .env
```

**Usage:**
```bash
cp .env.staging.example .env
docker-compose up
```

### Production - Kubernetes (.env.production.example)

```bash
# Kubernetes with Stork service discovery
REMOTE_CONTROL_SERVICE_USE_STORK=true
REMOTE_CONTROL_SERVICE_STORK_NAME=remote-control-service
REMOTE_CONTROL_SERVICE_USE_PLAINTEXT=false
REMOTE_CONTROL_SERVICE_LOAD_BALANCER=LEAST_REQUESTS
```

**Kubernetes deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        env:
        - name: REMOTE_CONTROL_SERVICE_USE_STORK
          value: "true"
        - name: REMOTE_CONTROL_SERVICE_STORK_NAME
          value: "remote-control-service"
        - name: REMOTE_CONTROL_SERVICE_USE_PLAINTEXT
          value: "false"
        - name: ZEQUENT_MAX_RETRY_ATTEMPTS
          value: "5"
```

## Code Usage

### With CDI (Recommended)

```java
@ApplicationScoped
public class MyService {

    @Inject
    ZequentClient client;  // Automatically configured from properties!

    public void doSomething() {
        var request = TakeoffRequest.builder()
            .sn("YOUR_DEVICE_SN")
            .latitude(47.3769f)
            .longitude(8.5417f)
            .altitude(100.0f)
            .build();

        var response = client.remoteControl().takeoff(request).join();
    }
}
```

### Standalone (Programmatic)

You can still use the builder if needed:

```java
ZequentClient client = ZequentClient.builder()
    .remoteControl()
        .host("custom-host")
        .port(8002)
        .done()
    .build();
```

## Load Balancer Types

| Type | Description | Use Case |
|------|-------------|----------|
| `ROUND_ROBIN` | Distribute requests evenly across instances | General purpose (default) |
| `RANDOM` | Random instance selection | Simple load distribution |
| `LEAST_REQUESTS` | Route to instance with fewest active requests | Optimized for varying request durations |
| `POWER_OF_TWO_CHOICES` | Pick best of 2 random instances | Balance between random and least-requests |

## Stork Service Discovery

### Kubernetes

```bash
REMOTE_CONTROL_SERVICE_USE_STORK=true
REMOTE_CONTROL_SERVICE_STORK_NAME=remote-control-service
```

Stork will automatically discover service endpoints via Kubernetes DNS.

### Consul

Add to `application.properties`:
```properties
stork.remote-control-service.service-discovery.type=consul
stork.remote-control-service.service-discovery.consul-host=consul.example.com
stork.remote-control-service.service-discovery.consul-port=8500
```

## Tips

### 1. Environment Switching
**Never change code!** Just switch `.env` files:
```bash
# Development
cp .env.dev.example .env && mvn quarkus:dev

# Staging
cp .env.staging.example .env && docker-compose up

# Production
# Set environment variables in K8s and deploy
```

### 2. Override Individual Settings
Mix defaults with overrides:
```bash
# Use .env defaults but override one service
REMOTE_CONTROL_SERVICE_HOST=custom-host mvn quarkus:dev
```

### 3. Validate Configuration
Check logs on startup:
```
Service 'remote-control' configured: host=localhost, port=8002, useStork=false
Service 'mission-autonomy' configured: host=localhost, port=8004, useStork=false
Service 'live-data' configured: host=localhost, port=8003, useStork=false
```

## Troubleshooting

### Problem: Services not connecting

**Check 1:** Verify environment variables
```bash
echo $REMOTE_CONTROL_SERVICE_HOST
echo $REMOTE_CONTROL_SERVICE_PORT
```

**Check 2:** Check application logs for configuration
```
Service 'remote-control' configured: host=..., port=...
```

**Check 3:** Test connectivity
```bash
telnet $REMOTE_CONTROL_SERVICE_HOST $REMOTE_CONTROL_SERVICE_PORT
```

### Problem: Stork not discovering services

**Check 1:** Verify Stork is enabled
```bash
echo $REMOTE_CONTROL_SERVICE_USE_STORK
# Should be: true
```

**Check 2:** Check Kubernetes service exists
```bash
kubectl get service remote-control-service
```

**Check 3:** Check Stork logs
```
Stork discovering service: remote-control-service
```

## Summary

 **No code changes** for environment switching
 **Properties or environment variables** - your choice
 **Multi-service** - each service independently configured
 **Stork support** - automatic service discovery
 **Load balancing** - per-service configuration
 **Resilience** - built-in retry, circuit breaker, timeouts

Just copy the right `.env` file and run! 
