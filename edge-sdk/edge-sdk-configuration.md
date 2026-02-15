# Edge SDK -- Configuration Guide

## Overview

The Zequent Edge SDK is configured primarily through Quarkus `application.properties` and environment variables. Configuration covers the edge identity, gRPC service endpoints, MQTT broker settings (for adapters that use MQTT), and operational parameters.

## Table of Contents

- [Configuration Methods](#configuration-methods)
- [Edge Identity Configuration](#edge-identity-configuration)
- [gRPC Client Configuration](#grpc-client-configuration)
- [gRPC Server Configuration](#grpc-server-configuration)
- [MQTT Configuration](#mqtt-configuration)
- [Object Storage Configuration](#object-storage-configuration)
- [Monitoring and Observability](#monitoring-and-observability)
- [Environment-Specific Examples](#environment-specific-examples)
- [Troubleshooting](#troubleshooting)

---

## Configuration Methods

### Priority Order (highest to lowest)

1. **Environment Variables** (e.g., `ZEQUENT_EDGE_SN`)
2. **System Properties** (e.g., `-Dzequent.edge.sn=...`)
3. **.env file** (automatically loaded by Quarkus)
4. **application.properties** (defaults)

Quarkus maps property names to environment variable names by converting to uppercase and replacing dots and hyphens with underscores. For example:

| Property | Environment Variable |
|----------|---------------------|
| `zequent.edge.sn` | `ZEQUENT_EDGE_SN` |
| `zequent.edge.asset-type` | `ZEQUENT_EDGE_ASSET_TYPE` |
| `quarkus.grpc.clients.live-data-service.host` | `QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST` |

---

## Edge Identity Configuration

These properties identify your edge adapter instance to the platform. They are mapped through the `EdgeClientConfig` interface using Quarkus `@ConfigMapping(prefix = "zequent.edge")`.

| Property | Environment Variable | Required | Default | Description |
|----------|---------------------|----------|---------|-------------|
| `zequent.edge.endpoint` | `ZEQUENT_EDGE_ENDPOINT` | Yes | -- | The address this adapter listens on (host:port) |
| `zequent.edge.sn` | `ZEQUENT_EDGE_SN` | Yes | -- | Serial number of the managed device |
| `zequent.edge.asset-type` | `ZEQUENT_EDGE_ASSET_TYPE` | Yes | -- | Asset type enum value (see below) |
| `zequent.edge.asset-vendor` | `ZEQUENT_EDGE_ASSET_VENDOR` | Yes | -- | Asset vendor enum value (see below) |
| `zequent.edge.timeout` | `ZEQUENT_EDGE_TIMEOUT` | No | `30s` | Command execution timeout (Duration) |
| `zequent.edge.max-retries` | `ZEQUENT_EDGE_MAX_RETRIES` | No | `3` | Maximum retry attempts for failed operations |
| `zequent.edge.asset-id` | `ZEQUENT_EDGE_ASSET_ID` | No | -- | Platform asset ID (set after registration) |

### Asset Type Values

| Value | Description |
|-------|-------------|
| `ASSET_TYPE_DOCK` | Docking station |
| `ASSET_TYPE_DRONE` | Standalone drone |
| `ASSET_TYPE_VEHICLE` | Ground vehicle |
| `ASSET_TYPE_RC` | Remote controller |
| `ASSET_TYPE_UNKNOWN` | Unknown type |

### Asset Vendor Values

| Value | Description |
|-------|-------------|
| `DJI` | DJI |
| `VENDOR_UNKNOWN` | Unknown vendor |

### Example

```properties
zequent.edge.endpoint=localhost:9001
zequent.edge.sn=YOUR_DEVICE_SN
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=DJI
zequent.edge.timeout=30s
zequent.edge.max-retries=3
```

---

## gRPC Client Configuration

The edge adapter connects to platform services (Live Data, Connector) via gRPC. Each client is configured using the standard Quarkus gRPC client properties.

### Live Data Service

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `quarkus.grpc.clients.live-data-service.host` | `QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST` | `localhost` | Hostname |
| `quarkus.grpc.clients.live-data-service.port` | `QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT` | `8003` | Port |
| `quarkus.grpc.clients.live-data-service.keep-alive-without-calls` | -- | `true` | Keep connection alive even without active RPCs |

### Connector Service

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `quarkus.grpc.clients.connector-service.host` | `QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_HOST` | `localhost` | Hostname |
| `quarkus.grpc.clients.connector-service.port` | `QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_PORT` | `8010` | Port |
| `quarkus.grpc.clients.connector-service.keep-alive-without-calls` | -- | `true` | Keep connection alive even without active RPCs |

### Example

```properties
# Local development
quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003
quarkus.grpc.clients.live-data-service.keep-alive-without-calls=true

quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010
quarkus.grpc.clients.connector-service.keep-alive-without-calls=true
```

---

## gRPC Server Configuration

The edge adapter itself exposes a gRPC server for receiving commands from the platform.

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `quarkus.http.host` | `QUARKUS_HTTP_HOST` | `0.0.0.0` | Bind address |
| `quarkus.http.port` | `QUARKUS_HTTP_PORT` | `9001` | HTTP/gRPC port |
| `quarkus.grpc.server.use-separate-server` | -- | `false` | Use the same port for HTTP and gRPC |

### Example

```properties
quarkus.http.host=0.0.0.0
quarkus.http.port=9001
quarkus.grpc.server.use-separate-server=false
```

---

## MQTT Configuration

Many edge adapters use MQTT to communicate with the physical device (e.g., DJI docks use MQTT for OSD telemetry, service commands, and state updates). MQTT is configured through the SmallRye Reactive Messaging MQTT connector.

### Broker Configuration

Custom properties for the broker connection (used by adapter-specific code):

| Property | Environment Variable | Description |
|----------|---------------------|-------------|
| `zequent.mqtt.broker.host` | `ZEQUENT_MQTT_BROKER_HOST` | MQTT broker hostname |
| `zequent.mqtt.broker.username` | `ZEQUENT_MQTT_BROKER_USERNAME` | MQTT username |
| `zequent.mqtt.broker.password` | `ZEQUENT_MQTT_BROKER_PASSWORD` | MQTT password |

### Channel Configuration Pattern

Each MQTT channel (incoming or outgoing) follows this pattern:

```properties
# Incoming channel
mp.messaging.incoming.<channel-name>.connector=smallrye-mqtt
mp.messaging.incoming.<channel-name>.topic=<mqtt/topic/pattern>
mp.messaging.incoming.<channel-name>.host=<broker-host>
mp.messaging.incoming.<channel-name>.port=8883
mp.messaging.incoming.<channel-name>.username=<username>
mp.messaging.incoming.<channel-name>.password=<password>
mp.messaging.incoming.<channel-name>.ssl=true

# Outgoing channel
mp.messaging.outgoing.<channel-name>.connector=smallrye-mqtt
mp.messaging.outgoing.<channel-name>.host=<broker-host>
mp.messaging.outgoing.<channel-name>.port=8883
mp.messaging.outgoing.<channel-name>.username=<username>
mp.messaging.outgoing.<channel-name>.password=<password>
mp.messaging.outgoing.<channel-name>.ssl=true
```

### Typical Channels for a DJI Adapter

| Channel Name | Direction | Topic Pattern | Purpose |
|-------------|-----------|---------------|---------|
| `osd` | Incoming | `thing/product/+/osd` | On-screen display / telemetry |
| `state` | Incoming | `thing/product/+/state` | Device state changes |
| `status` | Incoming | `sys/product/+/status` | Topology updates |
| `status_reply` | Outgoing | (dynamic) | Topology update replies |
| `cloud-to-dock` | Outgoing | (dynamic) | Commands sent to the dock |
| `services-reply` | Incoming | `thing/product/+/services_reply` | Replies to service commands |
| `requests` | Incoming | `thing/product/+/requests` | Device-initiated requests |
| `drc-up` | Incoming | `thing/product/+/drc/up` | DRC (Direct Remote Control) upstream data |

---

## Object Storage Configuration

If your adapter needs to upload files (e.g., KMZ flight plans) to object storage:

| Property | Environment Variable | Description |
|----------|---------------------|-------------|
| `storage.endpoint` | `STORAGE_ENDPOINT` | S3-compatible endpoint URL |
| `storage.access-key` | `STORAGE_ACCESS_KEY` | Access key |
| `storage.secret-key` | `STORAGE_SECRET_KEY` | Secret key |
| `storage.region` | `STORAGE_REGION` | Storage region |

### Example

```properties
storage.endpoint=https://s3.amazonaws.com
storage.access-key=YOUR_ACCESS_KEY
storage.secret-key=YOUR_SECRET_KEY
storage.region=eu-central-1
```

---

## Monitoring and Observability

| Property | Environment Variable | Default | Description |
|----------|---------------------|---------|-------------|
| `quarkus.micrometer.enabled` | -- | `true` | Enable Micrometer metrics |
| `quarkus.micrometer.export.prometheus.enabled` | -- | `true` | Enable Prometheus exporter |
| `quarkus.micrometer.export.prometheus.path` | -- | `/q/metrics` | Metrics endpoint path |
| `quarkus.otel.traces.enabled` | -- | `true` | Enable OpenTelemetry tracing |
| `quarkus.otel.metrics.enabled` | -- | `true` | Enable OpenTelemetry metrics |
| `quarkus.otel.logs.enabled` | -- | `true` | Enable OpenTelemetry log export |
| `quarkus.otel.exporter.otlp.endpoint` | `QUARKUS_OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | OTLP collector endpoint |

### Example

```properties
quarkus.micrometer.enabled=true
quarkus.micrometer.export.prometheus.enabled=true
quarkus.micrometer.export.prometheus.path=/q/metrics
quarkus.otel.traces.enabled=true
quarkus.otel.exporter.otlp.endpoint=http://localhost:4317
```

---

## Environment-Specific Examples

### Local Development

```properties
# application.properties
quarkus.http.port=9001

zequent.edge.endpoint=localhost:9001
zequent.edge.sn=YOUR_DEVICE_SN
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=DJI

quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003

quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010

quarkus.redis.hosts=redis://localhost:6379

quarkus.devservices.enabled=false
```

### Docker Compose

Override via environment variables in your compose file:

```yaml
services:
  edge-adapter:
    image: ghcr.io/zequent/edge-dji:latest
    ports:
      - "9001:9001"
    environment:
      - ZEQUENT_EDGE_ENDPOINT=edge-adapter:9001
      - ZEQUENT_EDGE_SN=YOUR_DEVICE_SN
      - QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST=live-data-service
      - QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT=8003
      - QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_HOST=connector-service
      - QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_PORT=8080
      - QUARKUS_REDIS_HOSTS=redis://redis:6379
```

### Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-dji
spec:
  template:
    spec:
      containers:
      - name: edge-dji
        image: ghcr.io/zequent/edge-dji:latest
        ports:
        - containerPort: 9001
        env:
        - name: ZEQUENT_EDGE_ENDPOINT
          value: "edge-dji:9001"
        - name: ZEQUENT_EDGE_SN
          valueFrom:
            secretKeyRef:
              name: edge-secrets
              key: device-sn
        - name: QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST
          value: "live-data-service"
        - name: QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_HOST
          value: "connector-service"
        - name: QUARKUS_REDIS_HOSTS
          value: "redis://redis:6379"
```

---

## Troubleshooting

### Problem: Adapter cannot connect to platform services

**Check 1:** Verify gRPC client settings:

```bash
echo $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST
echo $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT
```

**Check 2:** Test network connectivity:

```bash
nc -zv $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST $QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT
```

**Check 3:** Look for connection errors in the logs:

```
gRPC stream failed for device XXXXX: UNAVAILABLE
```

### Problem: Telemetry not arriving at the platform

**Check 1:** Verify the device serial number matches what is configured:

```bash
echo $ZEQUENT_EDGE_SN
```

**Check 2:** Look for telemetry stream logs:

```
Started gRPC telemetry stream for device XXXXX
Telemetry response received for device XXXXX
```

**Check 3:** Ensure the Live Data Service is running and reachable.

### Problem: MQTT messages not being received

**Check 1:** Verify MQTT broker connection:

```bash
echo $ZEQUENT_MQTT_BROKER_HOST
```

**Check 2:** Check that MQTT topics match the device model's expected patterns.

**Check 3:** Look for MQTT connection logs on startup.

### Problem: Configuration not taking effect

**Check 1:** Environment variables take precedence over `application.properties`. Verify no conflicting env vars are set.

**Check 2:** Quarkus caches configuration at startup. Restart the application after changing properties.

**Check 3:** Use Quarkus dev mode (`./mvnw quarkus:dev`) for live reload during development.
