# Zequent Edge SDK

The Edge SDK is used to build custom edge adapters that connect physical assets such as drones, docks, vehicles, and other remote devices to Zequent. A custom adapter exposes a consistent command and telemetry interface to the platform and to Client SDK consumers.

## Tech Specs

| Requirement | Version |
|-------------|---------|
| Java        | 25      |
| Maven       | 3.9.9+  |
| Quarkus     | 3.x     |
| gRPC        | via Quarkus gRPC extension |

## Overview

The Edge SDK sits between the physical device and the Zequent platform services. An edge adapter is a standalone Quarkus application that depends on the Edge SDK library and provides concrete implementations for the commands relevant to its hardware.


## Core Concepts

### EdgeClient

The central entry point of the SDK. It is a CDI-managed bean that gives you access to the configured serial number, edge configuration, and the `EdgeAdapterService` instance.

### EdgeAdapterService

The primary interface that every edge adapter must implement. It declares commands for flight control, dock management, camera operations, manual control, live streaming, task execution, and more. All methods come with a default `NOT_IMPLEMENTED` return so you only need to override what your hardware supports.

### ConnectorService

Provides access to the platform's Connector Service over gRPC: register and manage your asset(s), and look up missions, tasks, schedulers and organizations.

### LiveDataService

Manages persistent gRPC telemetry streams. It lets you push asset and sub-asset telemetry data to the Live Data Service using either the POJO-based API or the raw Proto-based API.

### MissionAutonomyService

Communicates with the Mission Autonomy Service over gRPC to look up scheduler definitions. Task execution itself is driven by the platform calling *into* your adapter (see `EdgeAdapterService`), not by the adapter polling this service. Missions and tasks are authored and triggered through the **Client SDK**, not the Edge SDK.

## Available Documentation

| Document | Description |
|----------|-------------|
| [DJI Adapter Deployment](edge-sdk-dji-adapter-deployment.md) | Deploy the DJI edge adapter container via Docker or Kubernetes |
| [Quickstart](edge-sdk-quickstart.md) | Get a new edge adapter project up and running in minutes |
| [Configuration](edge-sdk-configuration.md) | All configuration properties and environment variable mappings |
| [Edge Adapter](edge-sdk-adapter.md) | Implementing the EdgeAdapterService interface |
| [Live Data](edge-sdk-live-data.md) | Producing telemetry data streams |
| [Connector](edge-sdk-connector.md) | Asset and resource management via the Connector Service |
| [Mission Autonomy](edge-sdk-mission-autonomy.md) | Scheduler lookup, task lifecycle, and custom commands |
| [Models Reference](../api-reference/edge-sdk-models.md) | Request, response, and telemetry data model reference |
| [Go Edge SDK](edge-sdk-go-overview.md) | Go equivalent — an older API surface than this one, see its status note |

## Quick Start

Add the Edge SDK dependency to your project:

```xml
<dependency>
  <groupId>com.zqnt.sdk</groupId>
  <artifactId>edge-java-sdk</artifactId>
  <version>1.3.0</version>
</dependency>
```

Check your package registry for the latest published version.

Configure your edge in `application.properties`:

```properties
zequent.edge.endpoint=localhost:9001
zequent.edge.sn=YOUR_DEVICE_SERIAL_NUMBER
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=DJI
```

Implement the adapter interface:

```java
@ApplicationScoped
public class MyEdgeAdapter implements EdgeAdapterService {

    @Override
    public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
        // your hardware-specific takeoff logic
        return CompletableFuture.completedFuture(
            CommandResult.success("Takeoff initiated", request.getSn())
        );
    }

    // override only the commands your device supports
}
```

Build and run your adapter as a container image:

```bash
docker run --env-file .env -p 9001:9001 your-registry/my-edge-adapter:latest
```

For a complete walkthrough, see the [Quickstart Guide](edge-sdk-quickstart.md).
