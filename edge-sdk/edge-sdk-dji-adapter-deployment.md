# DJI Edge Adapter -- Deployment Guide

The DJI Edge Adapter connects DJI docking stations and their sub-assets (drones) to the Zequent platform. It communicates with the dock via MQTT and exposes a gRPC interface toward the platform services.

---

## Prerequisites

- Access to the Zequent container registry (`ghcr.io/zequent`)
- A running MQTT broker reachable by the adapter (e.g. HiveMQ Cloud)
- Running Zequent platform services: Connector Service, Live Data Service, Mission Autonomy Service
- Redis instance

---

## Environment Variables

### Required

| Variable | Description |
|----------|-------------|
| `ZEQUENT_EDGE_SN` | Serial number of the managed DJI dock |
| `MQTT_BROKER_HOST` | MQTT broker hostname |
| `MQTT_USERNAME` | MQTT username for cloud backend channels |
| `MQTT_PASSWORD` | MQTT password for cloud backend channels |
| `MQTT_DOCK_USERNAME` | MQTT username for direct dock communication |
| `MQTT_DOCK_PASSWORD` | MQTT password for direct dock communication |
| `QUARKUS_REDIS_HOSTS` | Redis connection URL (e.g. `redis://redis:6379`) |
| `CONNECTOR_SERVICE_HOST` | Hostname of the Connector Service |
| `LIVE_DATA_SERVICE_HOST` | Hostname of the Live Data Service |
| `MISSION_AUTONOMY_SERVICE_HOST` | Hostname of the Mission Autonomy Service |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `MQTT_BROKER_PORT` | `8883` | MQTT broker port |
| `CONNECTOR_SERVICE_PORT` | `8010` | Connector Service gRPC port |
| `LIVE_DATA_SERVICE_PORT` | `8003` | Live Data Service gRPC port |
| `MISSION_AUTONOMY_SERVICE_PORT` | `8004` | Mission Autonomy Service gRPC port |
| `EDGE_ADAPTER_TARGET_ENDPOINTS` | `edge-adapter-dji:9001` | Address at which this adapter is reachable by the platform |
| `S3_ENDPOINT` | `https://s3.amazonaws.com` | S3-compatible storage endpoint (required for mission file uploads) |
| `S3_ACCESS_KEY` | -- | S3 access key |
| `S3_SECRET_KEY` | -- | S3 secret key |
| `S3_REGION` | `eu-central-1` | S3 region |
| `S3_BUCKET` | `zqnt` | S3 bucket name |
| `S3_OBJECT_KEY_PREFIX` | `zqnt` | Prefix for stored objects |
| `S3_USERNAME` | -- | S3 user identifier |

### Monitoring (all disabled by default)

| Variable | Default | Description |
|----------|---------|-------------|
| `MICROMETER_ENABLED` | `false` | Enable Micrometer metrics |
| `PROMETHEUS_ENABLED` | `false` | Enable Prometheus `/q/metrics` endpoint |
| `OTEL_TRACES_ENABLED` | `false` | Enable OpenTelemetry tracing |
| `OTEL_METRICS_ENABLED` | `false` | Enable OpenTelemetry metrics |
| `OTEL_LOGS_ENABLED` | `false` | Enable OpenTelemetry log export |
| `OTEL_ENDPOINT` | `http://jaeger-all-in-one:4317` | OTLP collector endpoint |
| `OTEL_RESOURCE_ATTRIBUTES` | `service.name=edge-adapter-dji` | OTel resource attributes |

---

## Docker Compose

```yaml
services:
  edge-adapter-dji:
    image: ghcr.io/zequent/edge-adapter-dji:latest
    ports:
      - "9001:9001"
    environment:
      - ZEQUENT_EDGE_SN=YOUR_DOCK_SERIAL_NUMBER
      - EDGE_ADAPTER_TARGET_ENDPOINTS=edge-adapter-dji:9001
      - MQTT_BROKER_HOST=your-broker.example.com
      - MQTT_USERNAME=backend
      - MQTT_PASSWORD=secret
      - MQTT_DOCK_USERNAME=dock
      - MQTT_DOCK_PASSWORD=secret
      - CONNECTOR_SERVICE_HOST=connector-service
      - LIVE_DATA_SERVICE_HOST=live-data-service
      - MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service
      - QUARKUS_REDIS_HOSTS=redis://redis:6379
```

---

## Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: edge-adapter-dji
spec:
  replicas: 1
  selector:
    matchLabels:
      app: edge-adapter-dji
  template:
    metadata:
      labels:
        app: edge-adapter-dji
    spec:
      containers:
        - name: edge-adapter-dji
          image: ghcr.io/zequent/edge-adapter-dji:latest
          ports:
            - containerPort: 9001
          env:
            - name: ZEQUENT_EDGE_SN
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: dock-sn
            - name: MQTT_BROKER_HOST
              value: "your-broker.example.com"
            - name: MQTT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-username
            - name: MQTT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-password
            - name: MQTT_DOCK_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-username
            - name: MQTT_DOCK_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-password
            - name: CONNECTOR_SERVICE_HOST
              value: "connector-service"
            - name: LIVE_DATA_SERVICE_HOST
              value: "live-data-service"
            - name: MISSION_AUTONOMY_SERVICE_HOST
              value: "mission-autonomy-service"
            - name: QUARKUS_REDIS_HOSTS
              value: "redis://redis:6379"
---
apiVersion: v1
kind: Service
metadata:
  name: edge-adapter-dji
spec:
  selector:
    app: edge-adapter-dji
  ports:
    - port: 9001
      targetPort: 9001
```

---

## MQTT Topics

The adapter subscribes and publishes to the following MQTT topics. The `+` wildcard matches the device serial number.

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `thing/product/+/osd` | Incoming | Drone telemetry (OSD data) |
| `thing/product/+/state` | Incoming | Device state updates |
| `sys/product/+/status` | Incoming | Dock/drone topology updates |
| `thing/product/+/services_reply` | Incoming | Replies to service commands |
| `thing/product/+/requests` | Incoming | Device-initiated requests |
| `thing/product/+/drc/up` | Incoming | Direct Remote Control upstream data |
| Cloud-to-dock topics | Outgoing | Commands sent to the dock |
| Status reply topics | Outgoing | Topology update acknowledgements |

---

## Port

The adapter listens on port `9001` for incoming gRPC commands from the platform.
