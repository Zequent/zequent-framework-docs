# Zequent Framework Guide

This guide helps you set up and use the Zequent Framework with your applications.

## Prerequisites

- Java 25
- Maven 3.9.9 or higher
- Docker or Podman
- GitHub account with access to Zequent platform

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
            <groupId>com.zequent.framework.client.sdk</groupId>
            <artifactId>java-client-sdk</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/Zequent/zequent-framework</url>
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
  zequent_db:
    image: docker.io/library/postgres:latest
    environment:
      POSTGRES_DB: zequent_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - zequent_data:/var/lib/postgresql

  redis:
    image: docker.io/library/redis:latest
    command: [ "redis-server", "--appendonly", "yes" ]
    ports:
      - "6379:6379"

  jaeger-all-in-one:
    image: docker.io/jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"
      - "14268:14268"
      - "4318:4318"
      - "4317:4317"
      - "14250:14250"
      - "14269:14269"
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  connector-service:
    depends_on:
      - zequent_db
      - redis
    image: ghcr.io/zequent/connector-service:latest
    ports:
      - "8080:8080"
    environment:
      - QUARKUS_REDIS_HOSTS=redis://redis:6379
      - QUARKUS_DATASOURCE_JDBC_URL=jdbc:postgresql://zequent_db:5432/zequent_db
      - QUARKUS_DATASOURCE_USERNAME=postgres
      - QUARKUS_DATASOURCE_PASSWORD=postgres
      - QUARKUS_DATASOURCE_REACTIVE_URL=postgresql://zequent_db:5432/zequent_db
      - QUARKUS_DATASOURCE_DB_KIND=postgresql
      - QUARKUS_FLYWAY_ACTIVE=true
      - QUARKUS_FLYWAY_BASELINE_ON_MIGRATE=true
      - QUARKUS_FLYWAY_MIGRATE_AT_START=true
      - QUARKUS_FLYWAY_SCHEMAS=schema_zequent

  live-data-service:
    depends_on:
      - zequent_db
      - redis
    image: ghcr.io/zequent/live-data-service:latest
    ports:
      - "8003:8003"
    environment:
      - QUARKUS_REDIS_HOSTS=redis://redis:6379

  remote-control-service:
    depends_on:
      - zequent_db
      - redis
    image: ghcr.io/zequent/remote-control-service:latest
    ports:
      - "8002:8002"
    environment:
      - QUARKUS_REDIS_HOSTS=redis://redis:6379

  edge-dji-service:
    depends_on:
      - zequent_db
      - redis
    image: ghcr.io/zequent/edge-dji:latest
    ports:
      - "9001:9001"
    environment:
      - QUARKUS_REDIS_HOSTS=redis://redis:6379
      - QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_HOST=live-data-service
      - QUARKUS_GRPC_CLIENTS_LIVE_DATA_SERVICE_PORT=8003
      - QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_HOST=connector-service
      - QUARKUS_GRPC_CLIENTS_CONNECTOR_SERVICE_PORT=8080

volumes:
  zequent_data:
```

**2. Start the Services**

Open a terminal in the same directory as your `docker-compose.yml` and run:

```bash
# For Docker
docker-compose up -d

# For Podman
podman-compose up -d
```
