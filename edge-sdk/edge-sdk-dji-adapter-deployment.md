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
| `ZQNT_MQTT_BROKER_HOST` | MQTT broker hostname |
| `ZQNT_MQTT_USERNAME` | MQTT username for cloud backend channels |
| `ZQNT_MQTT_PASSWORD` | MQTT password for cloud backend channels |
| `ZQNT_MQTT_DOCK_USERNAME` | MQTT username for direct dock communication |
| `ZQNT_MQTT_DOCK_PASSWORD` | MQTT password for direct dock communication |
| `CONNECTOR_SERVICE_HOST` | Hostname of the Connector Service |
| `LIVE_DATA_SERVICE_HOST` | Hostname of the Live Data Service |
| `MISSION_AUTONOMY_SERVICE_HOST` | Hostname of the Mission Autonomy Service |

### Optional

| Variable | Default | Description |
|----------|---------|-------------|
| `ZQNT_MQTT_BROKER_PORT` | `8883` | MQTT broker port (TLS) |
| `CONNECTOR_SERVICE_PORT` | `8010` | Connector Service gRPC port |
| `LIVE_DATA_SERVICE_PORT` | `8003` | Live Data Service gRPC port |
| `MISSION_AUTONOMY_SERVICE_PORT` | `8004` | Mission Autonomy Service gRPC port |
| `EDGE_ADAPTER_TARGET_ENDPOINTS` | `edge-adapter-dji:9001` | Address at which this adapter is reachable by the platform |
| `QUARKUS_REDIS_HOSTS` | `redis://redis:6379` | Redis connection URL |
| `S3_ENDPOINT` | `https://s3.amazonaws.com` | S3-compatible storage endpoint |
| `S3_REGION` | `eu-central-1` | S3 region |
| `S3_BUCKET` | `zqnt` | S3 bucket name |
| `S3_OBJECT_KEY_PREFIX` | `zqnt` | Prefix for stored objects |
| `S3_USERNAME` | -- | S3 user identifier |
| `S3_ACCESS_KEY` | -- | S3 access key |
| `S3_SECRET_KEY` | -- | S3 secret key |

### Monitoring (all disabled by default)

| Variable | Default | Description |
|----------|---------|-------------|
| `MICROMETER_ENABLED` | `false` | Enable Micrometer metrics |
| `PROMETHEUS_ENABLED` | `false` | Enable Prometheus scrape endpoint |
| `PROMETHEUS_PATH` | `/q/metrics` | Prometheus scrape path |
| `MICROMETER_BINDER_JVM` | `false` | JVM metrics |
| `MICROMETER_BINDER_SYSTEM` | `false` | System metrics |
| `MICROMETER_BINDER_HTTP_SERVER_ENABLED` | `false` | HTTP server metrics |
| `MICROMETER_BINDER_GRPC_SERVER_ENABLED` | `false` | gRPC server metrics |
| `MICROMETER_BINDER_GRPC_CLIENT_ENABLED` | `false` | gRPC client metrics |
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
      - QUARKUS_PROFILE=docker
      - EDGE_ADAPTER_TARGET_ENDPOINTS=edge-adapter-dji:9001
      - ZQNT_MQTT_BROKER_HOST=your-broker.example.com
      - ZQNT_MQTT_BROKER_PORT=8883
      - ZQNT_MQTT_USERNAME=backend
      - ZQNT_MQTT_PASSWORD=secret
      - ZQNT_MQTT_DOCK_USERNAME=dock
      - ZQNT_MQTT_DOCK_PASSWORD=secret
      - CONNECTOR_SERVICE_HOST=connector-service
      - CONNECTOR_SERVICE_PORT=8010
      - LIVE_DATA_SERVICE_HOST=live-data-service
      - LIVE_DATA_SERVICE_PORT=8003
      - MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service
      - MISSION_AUTONOMY_SERVICE_PORT=8004
      - QUARKUS_REDIS_HOSTS=redis://redis:6379
      # S3 (optional - required for mission file uploads)
      - S3_ENDPOINT=https://s3.amazonaws.com
      - S3_REGION=eu-central-1
      - S3_BUCKET=zqnt
      - S3_USERNAME=<your-s3-username>
      - S3_ACCESS_KEY=<your-access-key>
      - S3_SECRET_KEY=<your-secret-key>
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
            - name: QUARKUS_PROFILE
              value: "k8s"
            - name: EDGE_ADAPTER_TARGET_ENDPOINTS
              value: "edge-adapter-dji"
            - name: ZQNT_MQTT_BROKER_HOST
              value: "your-broker.example.com"
            - name: ZQNT_MQTT_BROKER_PORT
              value: "8883"
            - name: ZQNT_MQTT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-username
            - name: ZQNT_MQTT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-password
            - name: ZQNT_MQTT_DOCK_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-username
            - name: ZQNT_MQTT_DOCK_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: mqtt-dock-password
            - name: CONNECTOR_SERVICE_HOST
              value: "connector-service"
            - name: CONNECTOR_SERVICE_PORT
              value: "8010"
            - name: LIVE_DATA_SERVICE_HOST
              value: "live-data-service"
            - name: LIVE_DATA_SERVICE_PORT
              value: "8003"
            - name: MISSION_AUTONOMY_SERVICE_HOST
              value: "mission-autonomy-service"
            - name: MISSION_AUTONOMY_SERVICE_PORT
              value: "8004"
            - name: QUARKUS_REDIS_HOSTS
              value: "redis://redis:6379"
            # S3 (optional - required for mission file uploads)
            - name: S3_ENDPOINT
              value: "https://s3.amazonaws.com"
            - name: S3_REGION
              value: "eu-central-1"
            - name: S3_BUCKET
              value: "zqnt"
            - name: S3_USERNAME
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-username
            - name: S3_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-access-key
            - name: S3_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: edge-adapter-dji-secrets
                  key: s3-secret-key
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

> **Note:** In Kubernetes mode the adapter uses Stork with Kubernetes service discovery for all upstream gRPC clients. The `CONNECTOR_SERVICE_HOST` / `LIVE_DATA_SERVICE_HOST` / `MISSION_AUTONOMY_SERVICE_HOST` variables are used for Docker Compose only. In the `k8s` profile, service names are resolved automatically via the Stork Kubernetes provider.

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

The adapter listens on port `9001` for incoming gRPC commands from the platform (HTTP and gRPC share the same port via `use-separate-server=false`).
