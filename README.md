# Zequent Framework Guide

This guide helps you set up and use the Zequent Framework with your applications.

The framework ships **two language editions** of each SDK. Pick the one that
matches your stack:

| SDK              | Java                                                     | Python                                                                |
|------------------|----------------------------------------------------------|-----------------------------------------------------------------------|
| Edge SDK         | [edge-sdk-overview.md](edge-sdk/edge-sdk-overview.md)    | [edge-sdk-python-overview.md](edge-sdk/edge-sdk-python-overview.md)   |
| Client SDK       | [QUICKSTART.md](client-sdk/QUICKSTART.md)                | [QUICKSTART_PYTHON.md](client-sdk/QUICKSTART_PYTHON.md)               |

The Python SDKs target Python 3.12+, use `uv` for dependency management, and
expose the same gRPC contract as the Java SDKs — your platform services don't
care which language an adapter or client is written in.

## Prerequisites

### Java track

- Java 17 or higher
- Maven 3.6.X or higher
- Docker, Podman or Kubernetes*
- GitHub Account to acces public Packages and Container Images

### Python track

- Python 3.12 or higher
- [`uv`](https://docs.astral.sh/uv/) (recommended) or `pip`
- Docker, Podman or Kubernetes*
- GitHub Account to access private packages (when applicable)

## Setup

### 1. Configure Maven

First, configure Maven to access GitHub Packages. Add the following to your `~/.m2/settings.xml` file. This allows Maven to download the necessary Zequent Framework dependencies.

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>YOUR_GITHUB_USERNAME</username>
      <password>YOUR_GITHUB_TOKEN</password>
    </server>
  </servers>
</settings>
```

> **Note:** Replace `YOUR_GITHUB_USERNAME` with your GitHub username and `YOUR_GITHUB_TOKEN` with a personal access token that has `read:packages` scope.

### 2. Set Up Your Project

Next, set up a new Maven project or edit existing Maven projects to add the Zequent Framework dependency..

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-zequent-app</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.zqnt.sdk</groupId>
            <artifactId>client-sdk</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/Zequent/zqnt-client-sdk-java</url>
        </repository>
    </repositories>
</project>
```

### 3. Run Services

The Zequent Framework relies on several services. You can run them easily using Docker or Podman.

**1. Create `docker-compose.yml`**

Create a `docker-compose.yml` file in your project's root directory:

```yaml
version: "3.8"
services:
  # ==================== INFRASTRUCTURE ====================
  zequent_db:
    image: docker.io/timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_DB: zequent_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - zequent_data:/var/lib/timescaledb/data
    command: postgres -c shared_preload_libraries=timescaledb


  redis:
    image: docker.io/library/redis:latest
    container_name: redis
    command: ["redis-server", "--appendonly", "yes"]
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped


  # ==================== OBSERVABILITY (Optional) ====================
  jaeger-all-in-one:
    image: docker.io/jaegertracing/all-in-one:latest
    container_name: jaeger-all-in-one
    ports:
      - "16686:16686" # Jaeger UI
      - "14268:14268" # Receive legacy OpenTracing traces, optional
      - "4318:4318"   # OTLP HTTP receiver
      - "4317:4317"   # OTLP gRPC
      - "14250:14250" # Receive from external otel-collector, optional
      - "14269:14269" # Metrics Reciever
    environment:
      - JAEGER_UI_CONFIG={"theme":"light"}
      - COLLECTOR_OTLP_ENABLED=true


  prometheus:
    image: docker.io/prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus-local.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    extra_hosts:
      - "host.containers.internal:host-gateway"


  grafana:
    image: docker.io/grafana/grafana:latest
    container_name: grafana
    depends_on:
      - prometheus
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=zequent2024
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://localhost:3000
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro

  # ==================== CORE SERVICES ====================
  connector-service:
    image: ghcr.io/zequent/connector-service:latest
    pull_policy: always
    container_name: connector-service
    depends_on:
      - zequent_db
      - redis
    ports:
      - "${CONNECTOR_PORT:-8010}:8010"
    environment:
      # Profile
      - QUARKUS_PROFILE=docker
      # Redis
      - REDIS_URL=redis://redis:6379
      # Database
      - DATABASE_URL=jdbc:postgresql://zequent_db:5432/zequent_db
      - DATABASE_REACTIVE_URL=postgresql://zequent_db:5432/zequent_db
      - DATABASE_USER=postgres
      - DATABASE_PASSWORD=postgres

      # OpenTelemetry (Optional - falls back to disabled if jaeger not running)
      - OTEL_TRACES_ENABLED=true
      - OTEL_METRICS_ENABLED=true
      - OTEL_LOGS_ENABLED=true
      - OTEL_ENDPOINT=http://jaeger-all-in-one:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=connector-service
    restart: unless-stopped
    dns:
      - 8.8.8.8
      - 8.8.4.4


  live-data-service:
    image: ghcr.io/zequent/live-data-service:latest
    pull_policy: always
    container_name: live-data-service
    depends_on:
      - redis
      - connector-service
    ports:
      - "8003:8003"
    environment:
      # Profile
      - QUARKUS_PROFILE=docker

      # Redis
      - REDIS_URL=redis://redis:6379

      # gRPC Clients (uses Stork with static discovery)
      - CONNECTOR_SERVICE_HOST=connector-service
      - CONNECTOR_SERVICE_PORT=8010

      - OTEL_TRACES_ENABLED=true
      - OTEL_METRICS_ENABLED=true
      - OTEL_LOGS_ENABLED=true
      - OTEL_ENDPOINT=http://jaeger-all-in-one:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=live-data-service
    restart: unless-stopped
    dns:
      - 8.8.8.8
      - 8.8.4.4

  remote-control-service:
    image: ghcr.io/zequent/remote-control-service:latest
    pull_policy: always
    container_name: remote-control-service
    depends_on:
      - redis
      - live-data-service
      - connector-service
    ports:
      - "8002:8002"
    environment:
      # Profile
      - QUARKUS_PROFILE=docker

      # Redis
      - QUARKUS_REDIS_HOSTS=redis://redis:6379

      # gRPC Clients (uses Stork with environment variables from properties)
      - LIVE_DATA_SERVICE_HOST=live-data-service
      - LIVE_DATA_SERVICE_PORT=8003
      - CONNECTOR_SERVICE_HOST=connector-service
      - CONNECTOR_SERVICE_PORT=8010

      # OpenTelemetry (Optional - falls back to disabled if jaeger not running)
      - OTEL_TRACES_ENABLED=true
      - OTEL_METRICS_ENABLED=true
      - OTEL_LOGS_ENABLED=true
      - OTEL_ENDPOINT=http://jaeger-all-in-one:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=remote-control-service
    restart: unless-stopped
    dns:
      - 8.8.8.8
      - 8.8.4.4


  mission-autonomy-service:
    image: ghcr.io/zequent/mission-autonomy-service:latest
    pull_policy: always
    container_name: mission-autonomy-service
    depends_on:
      - redis
      - connector-service
    ports:
      - "8004:8004"
    environment:
      # Profile
      - QUARKUS_PROFILE=docker
      # Redis
      - REDIS_URL=redis://redis:6379
      # gRPC Clients (uses Stork with static discovery)
      - CONNECTOR_SERVICE_HOST=connector-service
      - CONNECTOR_SERVICE_PORT=8010
      # OpenTelemetry (Optional - falls back to disabled if jaeger not running)
      - OTEL_TRACES_ENABLED=true
      - OTEL_METRICS_ENABLED=true
      - OTEL_LOGS_ENABLED=true
      - OTEL_ENDPOINT=http://jaeger-all-in-one:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=mission-autonomy-service
    restart: unless-stopped
    dns:
      - 8.8.8.8
      - 8.8.4.4


  # ==================== EDGE SERVICES ====================
  edge-adapter-dji:
    image: ghcr.io/zequent/zqnt-edge-adapter-dji:latest
    pull_policy: always
    depends_on:
      - redis
      - live-data-service
      - connector-service
    ports:
      - "9001:9001"
    environment:
      # Profile
      - QUARKUS_PROFILE=docker

      - EDGE_ADAPTER_TARGET_ENDPOINTS=edge-adapter-dji:9001
      # Redis
      - QUARKUS_REDIS_HOSTS=redis://redis:6379

      # gRPC Clients
      - LIVE_DATA_SERVICE_HOST=live-data-service
      - LIVE_DATA_SERVICE_PORT=8003
      - CONNECTOR_SERVICE_HOST=connector-service
      - CONNECTOR_SERVICE_PORT=8010

      # OpenTelemetry (Optional - falls back to disabled if jaeger not running)
      - OTEL_TRACES_ENABLED=true
      - OTEL_METRICS_ENABLED=true
      - OTEL_LOGS_ENABLED=true
      - OTEL_ENDPOINT=http://jaeger-all-in-one:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=edge-adapter-dji
    restart: unless-stopped
    dns:
      - 8.8.8.8
      - 8.8.4.4


volumes:
  zequent_data:
  redis_data:
  prometheus_data:
  grafana_data:

```

**2. Start the Services**

Open a terminal in the same directory as your `docker-compose.yml` and run:

```bash
# For Docker
docker-compose up -d

# For Podman
podman-compose up -d
```


# Environment Configuration Guide

This guide provides a complete overview of all configurable environment variables for the Zequent Framework. Use this reference to customize your deployment for development, Docker Compose, or Kubernetes environments.

## Quick Start

1. Copy the example file:
   ```bash
   cp .env.custom.example .env.custom
   ```

2. Edit `.env.custom` with your deployment-specific values

3. Start the services:
   ```bash
   podman-compose --env-file .env.custom up
   ```

---

## Deployment Profile

| Variable | Description | Values | Default |
|----------|-------------|--------|---------|
| `QUARKUS_PROFILE` | Active runtime profile | `dev`, `docker`, `k8s` | `docker` |

**Profile Overview:**
- **`dev`**: Local development with localhost connections
- **`docker`**: Docker Compose with service discovery
- **`k8s`**: Kubernetes with native service discovery

---

## Infrastructure

### Redis Configuration

| Variable | Description | Example |
|----------|-------------|---------|
| `REDIS_URL` | Redis connection URL | `redis://redis:6379` |

**Environment-specific examples:**
```bash
# Docker Compose
REDIS_URL=redis://redis:6379

# Kubernetes
REDIS_URL=redis://redis.zequent-prod.svc.cluster.local:6379

# Local Development
REDIS_URL=redis://localhost:6379
```

### Database Configuration

> **Note:** Database configuration applies only to the **Connector Service**. Other services do not require database access.

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | JDBC connection URL | `jdbc:postgresql://zequent_db:5432/zequent_db` |
| `DATABASE_REACTIVE_URL` | Reactive SQL client URL | `postgresql://zequent_db:5432/zequent_db` |
| `DATABASE_USER` | Database username | `postgres` |
| `DATABASE_PASSWORD` | Database password | `postgres` |

**Production example:**
```bash
DATABASE_URL=jdbc:postgresql://timescaledb.production:5432/zequent_production
DATABASE_REACTIVE_URL=postgresql://timescaledb.production:5432/zequent_production
DATABASE_USER=zequent_app
DATABASE_PASSWORD=<secure-password>
```

---

## Service Discovery (gRPC)

Configure hostnames and ports for inter-service communication. Supports custom container/service names for multi-tenant or namespace-specific deployments.

| Service | Host Variable | Port Variable | Default Host | Port | Description |
|---------|--------------|---------------|--------------|------|-------------|
| **Connector** | `CONNECTOR_SERVICE_HOST` | `CONNECTOR_SERVICE_PORT` | `connector-service` | `8010` | Core asset & vendor management |
| **Live Data** | `LIVE_DATA_SERVICE_HOST` | `LIVE_DATA_SERVICE_PORT` | `live-data-service` | `8003` | Real-time telemetry streaming |
| **Remote Control** | `REMOTE_CONTROL_SERVICE_HOST` | `REMOTE_CONTROL_SERVICE_PORT` | `remote-control-service` | `8002` | Asset command execution |
| **Mission Autonomy** | `MISSION_AUTONOMY_SERVICE_HOST` | `MISSION_AUTONOMY_SERVICE_PORT` | `mission-autonomy-service` | `8004` | Mission planning & execution |

**Standard configuration:**
```bash
CONNECTOR_SERVICE_HOST=connector-service
CONNECTOR_SERVICE_PORT=8010
LIVE_DATA_SERVICE_HOST=live-data-service
LIVE_DATA_SERVICE_PORT=8003
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=8002
MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service
MISSION_AUTONOMY_SERVICE_PORT=8004
```

**Custom naming example:**
```bash
# Multi-tenant deployment
CONNECTOR_SERVICE_HOST=acme-connector
LIVE_DATA_SERVICE_HOST=acme-livedata
REMOTE_CONTROL_SERVICE_HOST=acme-remotecontrol
MISSION_AUTONOMY_SERVICE_HOST=acme-missions
```

**Kubernetes with namespace:**
```bash
# Full qualified domain names
CONNECTOR_SERVICE_HOST=connector-service.zequent-prod.svc.cluster.local
LIVE_DATA_SERVICE_HOST=live-data-service.zequent-prod.svc.cluster.local
REMOTE_CONTROL_SERVICE_HOST=remote-control-service.zequent-prod.svc.cluster.local
MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service.zequent-prod.svc.cluster.local
```

---

## gRPC Client-Side Load Balancing

Enables automatic load balancing across scaled service instances using DNS-based service discovery.

| Variable | Description | Values | Default |
|----------|-------------|--------|---------|
| `GRPC_LOAD_BALANCING_POLICY` | Load balancing algorithm | `round_robin`, `pick_first` | `round_robin` |
| `GRPC_ENABLE_DNS_RESOLVER` | Enable DNS-based service discovery | `true`, `false` | `true` |

**How it works:**
1. **DNS Resolution**: Service names resolve to all available pod/container IPs
2. **Client-Side Distribution**: gRPC client distributes requests across all instances
3. **Automatic Scaling**: New instances are automatically discovered via DNS

**Scaling example:**
```bash
# Scale connector service to 3 instances
docker-compose up --scale connector-service=3

# Kubernetes scaling
kubectl scale deployment connector-service --replicas=3
```

> **Important:** All scaled instances must use the same port. Load balancing happens at the gRPC client level, not at the service level.

---

## Observability

### OpenTelemetry (Distributed Tracing)

OpenTelemetry provides distributed tracing, metrics export, and structured logging for the entire microservices stack.

| Variable | Description | Default | Impact |
|----------|-------------|---------|--------|
| `OTEL_TRACES_ENABLED` | Enable distributed tracing | `true` | Traces gRPC calls, HTTP requests, database queries |
| `OTEL_METRICS_ENABLED` | Export metrics via OTLP | `true` | Exports performance metrics to collector |
| `OTEL_LOGS_ENABLED` | Forward logs via OTLP | `true` | Structured logs to centralized system |
| `OTEL_ENDPOINT` | OTLP collector endpoint | `http://jaeger-all-in-one:4317` | Target for telemetry data |

**Visualize traces in Jaeger:**
- Access Jaeger UI: `http://localhost:16686`
- View request flows across all services
- Identify performance bottlenecks
- Debug distributed transactions

**Production example:**
```bash
OTEL_TRACES_ENABLED=true
OTEL_METRICS_ENABLED=true
OTEL_LOGS_ENABLED=true
OTEL_ENDPOINT=http://otel-collector.monitoring.svc.cluster.local:4317
```

**Development (minimal overhead):**
```bash
OTEL_TRACES_ENABLED=false
OTEL_METRICS_ENABLED=false
OTEL_LOGS_ENABLED=false
```

### Micrometer Metrics

Micrometer provides JVM, HTTP, and gRPC metrics with Prometheus export.

| Variable | Description | Default | Endpoint |
|----------|-------------|---------|----------|
| `MICROMETER_ENABLED` | Enable Micrometer metrics collection | `true` | - |
| `PROMETHEUS_ENABLED` | Expose Prometheus metrics endpoint | `true` | `/q/metrics` |
| `PROMETHEUS_PATH` | Custom metrics endpoint path | `/q/metrics` | - |

**Detailed metrics binders (optional):**
```bash
MICROMETER_BINDER_JVM=true           # JVM memory, GC, threads
MICROMETER_BINDER_SYSTEM=true        # CPU, file descriptors, uptime
HTTP_SERVER_ENABLED=true             # HTTP request/response metrics
GRPC_SERVER_ENABLED=true             # gRPC server call metrics
GRPC_CLIENT_ENABLED=true             # gRPC client call metrics
```

**Access Prometheus metrics:**
```bash
# Connector Service
curl http://localhost:8010/q/metrics

# Live Data Service
curl http://localhost:8003/q/metrics
```

**Grafana dashboards:**
- Pre-configured dashboards in `monitoring/grafana/dashboards/`
- Access Grafana: `http://localhost:3000` (admin/zequent2024)

---

## Complete Configuration Examples

### Production (Full Observability)

```bash
# Profile
QUARKUS_PROFILE=docker

# Infrastructure
REDIS_URL=redis://redis.production:6379
DATABASE_URL=jdbc:postgresql://timescaledb.production:5432/zequent_db
DATABASE_REACTIVE_URL=postgresql://timescaledb.production:5432/zequent_db
DATABASE_USER=zequent_app
DATABASE_PASSWORD=<secure-password>

# Service Discovery
CONNECTOR_SERVICE_HOST=connector-service
CONNECTOR_SERVICE_PORT=8010
LIVE_DATA_SERVICE_HOST=live-data-service
LIVE_DATA_SERVICE_PORT=8003
REMOTE_CONTROL_SERVICE_HOST=remote-control-service
REMOTE_CONTROL_SERVICE_PORT=8002
MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service
MISSION_AUTONOMY_SERVICE_PORT=8004

# Load Balancing
GRPC_LOAD_BALANCING_POLICY=round_robin
GRPC_ENABLE_DNS_RESOLVER=true

# Observability
OTEL_TRACES_ENABLED=true
OTEL_METRICS_ENABLED=true
OTEL_LOGS_ENABLED=true
OTEL_ENDPOINT=http://otel-collector:4317
MICROMETER_ENABLED=true
PROMETHEUS_ENABLED=true
```

### Development (Minimal Overhead)

```bash
# Profile
QUARKUS_PROFILE=dev

# Observability (disabled for performance)
OTEL_TRACES_ENABLED=false
OTEL_METRICS_ENABLED=false
OTEL_LOGS_ENABLED=false
MICROMETER_ENABLED=false
PROMETHEUS_ENABLED=false
```

> **Note:** In `dev` profile, services automatically use localhost defaults. No infrastructure variables needed.

### Kubernetes with Custom Namespace

```bash
# Profile
QUARKUS_PROFILE=k8s

# Infrastructure
REDIS_URL=redis://redis.zequent-prod.svc.cluster.local:6379
DATABASE_URL=jdbc:postgresql://timescaledb.zequent-prod.svc.cluster.local:5432/zequent_db
DATABASE_REACTIVE_URL=postgresql://timescaledb.zequent-prod.svc.cluster.local:5432/zequent_db
DATABASE_USER=zequent_app
DATABASE_PASSWORD=<from-secret>

# Service Discovery (FQDN)
CONNECTOR_SERVICE_HOST=connector-service.zequent-prod.svc.cluster.local
LIVE_DATA_SERVICE_HOST=live-data-service.zequent-prod.svc.cluster.local
REMOTE_CONTROL_SERVICE_HOST=remote-control-service.zequent-prod.svc.cluster.local
MISSION_AUTONOMY_SERVICE_HOST=mission-autonomy-service.zequent-prod.svc.cluster.local

# Load Balancing
GRPC_LOAD_BALANCING_POLICY=round_robin
GRPC_ENABLE_DNS_RESOLVER=true

# Observability
OTEL_TRACES_ENABLED=true
OTEL_METRICS_ENABLED=true
OTEL_LOGS_ENABLED=true
OTEL_ENDPOINT=http://otel-collector.monitoring.svc.cluster.local:4317
MICROMETER_ENABLED=true
PROMETHEUS_ENABLED=true
```

---

## Container Port Overrides (Optional)

Override default container ports to avoid conflicts with other services on the host.

```bash
CONNECTOR_PORT=8010
LIVE_DATA_PORT=8003
REMOTE_CONTROL_PORT=8002
MISSION_AUTONOMY_PORT=8004
```

**Example with custom ports:**
```yaml
# podman-compose.yml
services:
  connector-service:
    ports:
      - "${CONNECTOR_PORT:-8010}:8010"
```

---

## Important Notes

### 1. Profile-Specific Behavior

- **`dev` profile**: Most environment variables are ignored. Services use localhost defaults defined in `application.properties`.
- **`docker` profile**: Uses service names for DNS resolution (e.g., `redis`, `connector-service`).
- **`k8s` profile**: Supports both short names (`redis`) and fully qualified domain names (`redis.namespace.svc.cluster.local`).

### 2. Load Balancing

Load balancing works automatically in Docker Compose and Kubernetes when:
- Multiple replicas/pods are running
- `GRPC_ENABLE_DNS_RESOLVER=true`
- All instances use the same port

### 3. Redis Configuration

Ensure consistency across all services. Most services use `REDIS_URL=redis://redis:6379`.

### 4. Security Considerations

- **Never commit** `.env.custom` files with production credentials
- Use secrets management (Kubernetes Secrets, HashiCorp Vault) for sensitive data
- Rotate database passwords regularly
- Use TLS for Redis and PostgreSQL in production

### 5. Monitoring Stack

The framework includes a complete monitoring stack:
- **Jaeger**: Distributed tracing (`http://localhost:16686`)
- **Prometheus**: Metrics collection (`http://localhost:9090`)
- **Grafana**: Visualization dashboards (`http://localhost:3000`)

Enable observability in production to monitor service health and performance.

---

## Troubleshooting

### Service Cannot Resolve Redis

**Symptom**: Service fails to connect to Redis with DNS resolution errors.

**Solution**: Ensure `REDIS_URL` uses the correct format:
```bash
# Correct
REDIS_URL=redis://redis:6379

# Incorrect (missing redis:// scheme)
REDIS_URL=redis:6379
```

### gRPC Load Balancing Not Working

**Symptom**: Requests always go to the same service instance.

**Solution**: Verify DNS resolver is enabled:
```bash
GRPC_ENABLE_DNS_RESOLVER=true
GRPC_LOAD_BALANCING_POLICY=round_robin
```

### Observability Data Not Appearing

**Symptom**: No traces in Jaeger or metrics in Prometheus.

**Solution**: Check that observability is enabled:
```bash
OTEL_TRACES_ENABLED=true
MICROMETER_ENABLED=true
PROMETHEUS_ENABLED=true
```

Verify the OTLP endpoint is reachable:
```bash
curl http://jaeger-all-in-one:4317
```

---

## Related Documentation

- [gRPC Configuration Guide](GRPC_CONFIGURATION.md)
- [Load Balancing Guide](LOAD_BALANCING_GUIDE.md)
- [Docker Compose Configuration](podman-compose.yml)
- [Application Properties](services/*/src/main/resources/application.properties)

---

**Last Updated**: 2025-02-22
**Framework Version**: Latest
