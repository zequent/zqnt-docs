# Edge SDK -- Quickstart Guide

This guide walks you through creating a new edge adapter project from scratch using the Zequent Edge SDK. By the end, you will have an adapter application that can be packaged as a container image, receive commands from the platform, and push telemetry data.

## Prerequisites

- Java 25
- Maven 3.9.9 or higher
- Docker or Podman (for running platform services)
- GitHub account with access to the Zequent packages repository

---

## Step 1: Create a New Quarkus Project

```bash
mvn io.quarkus:quarkus-maven-plugin:3.17.4:create \
    -DprojectGroupId=com.example.edge \
    -DprojectArtifactId=my-edge-adapter \
    -Dextensions="grpc,smallrye-health,arc"

cd my-edge-adapter
```

---

## Step 2: Add the Edge SDK Dependency

Open `pom.xml` and add the Edge SDK and the GitHub Packages repository:

```xml
<dependencies>
    <!-- Zequent Edge SDK -->
    <dependency>
        <groupId>com.zqnt.sdk</groupId>
        <artifactId>edge-java-sdk</artifactId>
        <version>1.3.0</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>github</id>
        <url>https://maven.pkg.github.com/Zequent/zqnt-edge-sdk-java</url>
    </repository>
</repositories>
```

Check your package registry for the latest published version.

Make sure your `~/.m2/settings.xml` has the GitHub credentials:

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

---

## Step 3: Configure the Adapter

Create or edit `src/main/resources/application.properties`:

```properties
# Application
quarkus.application.name=my-edge-adapter
quarkus.http.host=0.0.0.0
quarkus.http.port=9001

# Edge Identity
zequent.edge.endpoint=localhost:9001
zequent.edge.sn=YOUR_DEVICE_SERIAL_NUMBER
zequent.edge.asset-type=ASSET_TYPE_DOCK
zequent.edge.asset-vendor=DJI

# Platform Services
quarkus.grpc.clients.live-data-service.host=localhost
quarkus.grpc.clients.live-data-service.port=8003
quarkus.grpc.clients.live-data-service.keep-alive-without-calls=true

quarkus.grpc.clients.connector-service.host=localhost
quarkus.grpc.clients.connector-service.port=8010
quarkus.grpc.clients.connector-service.keep-alive-without-calls=true

# gRPC server shares HTTP port
quarkus.grpc.server.use-separate-server=false
```

---

## Step 4: Implement the EdgeAdapterService

Create your adapter class. You only need to override the commands that your hardware supports:

```java
package com.example.edge;

import com.zqnt.sdk.edge.adapter.application.EdgeAdapterService;
import com.zqnt.sdk.edge.adapter.domains.*;
import jakarta.enterprise.context.ApplicationScoped;
import lombok.extern.slf4j.Slf4j;
import java.util.concurrent.CompletableFuture;

@Slf4j
@ApplicationScoped
public class MyDeviceAdapter implements EdgeAdapterService {

    @Override
    public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
        log.info("Takeoff requested for SN: {}", request.getSn());

        // Replace with your actual device API call
        boolean success = true;

        if (success) {
            return CompletableFuture.completedFuture(
                CommandResult.success("Takeoff initiated", request.getTid(), request.getSn())
            );
        } else {
            return CompletableFuture.completedFuture(
                CommandResult.error("Takeoff failed", request.getSn())
            );
        }
    }

    @Override
    public CompletableFuture<CommandResult> returnToHome(ReturnToHomeRequest request) {
        log.info("Return to home for SN: {}", request.getSn());
        return CompletableFuture.completedFuture(
            CommandResult.success("Returning to home", request.getTid(), request.getSn())
        );
    }

    @Override
    public CompletableFuture<CommandResult> openCover(String sn) {
        log.info("Opening cover for SN: {}", sn);
        return CompletableFuture.completedFuture(
            CommandResult.success("Cover opening", sn)
        );
    }

    // All other commands default to NOT_IMPLEMENTED and that is fine.
    // Override more as your hardware supports them.
}
```

---

## Step 5: Push Telemetry Data (Optional)

If your device produces telemetry, inject the `LiveDataService` and push data:

```java
package com.example.edge;

import com.zqnt.sdk.edge.livedata.application.LiveDataService;
import com.zqnt.sdk.edge.adapter.domains.TelemetryRequestData;
import com.zqnt.utils.edge.sdk.domains.TelemetryData;
import jakarta.enterprise.context.ApplicationScoped;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@ApplicationScoped
public class TelemetryProducer {

    private final LiveDataService liveDataService;

    public TelemetryProducer(LiveDataService liveDataService) {
        this.liveDataService = liveDataService;
    }

    public void sendAssetTelemetry(String sn) {
        TelemetryData.AssetDetails assetDetails = TelemetryData.AssetDetails.builder()
            .environmentTemp(22.5f)
            .humidity(65.0f)
            .build();

        TelemetryData telemetry = TelemetryData.builder()
            .id(UUID.randomUUID().toString())
            .timestamp(LocalDateTime.now())
            .sn(sn)
            .latitude(47.3769)
            .longitude(8.5417)
            .absoluteAltitude(450.0f)
            .asset(assetDetails)
            .build();

        TelemetryRequestData data = TelemetryRequestData.builder()
            .sn(sn)
            .tid(UUID.randomUUID().toString())
            .timestamp(LocalDateTime.now())
            .telemetry(telemetry)
            .build();

        liveDataService.produceTelemetryData(data)
            .thenRun(() -> log.debug("Telemetry sent for {}", sn))
            .exceptionally(err -> {
                log.error("Telemetry error", err);
                return null;
            });
    }
}
```

---

## Step 6: Start Platform Services

Before running your adapter, start the required platform services from the published container images:

```bash
docker compose -f docker-compose.customer.yml up -d
```

The compose file uses one deployment-local `.env` file through `env_file`.

---

## Step 7: Run Your Adapter

```bash
docker run --env-file .env -p 9001:9001 your-registry/my-edge-adapter:latest
```

You should see output similar to:

```
Listening on: http://0.0.0.0:9001
gRPC Server started on 0.0.0.0:9001
```

Your adapter is now running and ready to receive commands from the platform via gRPC.

---

## Step 8: Test with grpcurl

You can test your adapter endpoint with `grpcurl`:

```bash
# Install grpcurl if needed
# brew install grpcurl  (macOS)
# go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest  (Go)

# List available services
grpcurl -plaintext localhost:9001 list

# Call takeOff
grpcurl -plaintext -d '{
  "base": {
    "sn": "YOUR_DEVICE_SN",
    "tid": "test-123",
    "timestamp": {"seconds": 1700000000}
  },
  "request": {
    "latitude": 47.3769,
    "longitude": 8.5417,
    "altitude": 100.0
  }
}' localhost:9001 zqnt.EdgeAdapterService/TakeOff
```

---

## Project Structure

After completing this guide, your project should look like this:

```
my-edge-adapter/
  pom.xml
  src/
    main/
      java/
        com/example/edge/
          MyDeviceAdapter.java        # Your EdgeAdapterService implementation
          TelemetryProducer.java      # Optional: telemetry push logic
      resources/
        application.properties        # Configuration
  docker-compose.yml                  # Platform services (dev)
```

---

## Next Steps

- [Edge Adapter Reference](edge-sdk-adapter.md) -- Full command reference and advanced patterns
- [Configuration Guide](edge-sdk-configuration.md) -- All configuration properties
- [Live Data](edge-sdk-live-data.md) -- In-depth telemetry streaming guide
- [Connector](edge-sdk-connector.md) -- Asset and mission management
- [Models Reference](../api-reference/edge-sdk-models.md) -- Complete model documentation

For a ready-made DJI deployment, see [DJI Adapter Deployment](edge-sdk-dji-adapter-deployment.md).
