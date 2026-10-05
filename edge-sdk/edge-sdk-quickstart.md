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
        <version>2.0.0</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>github</id>
        <url>https://maven.pkg.github.com/Zequent/zqnt-edge-sdk-java</url>
    </repository>
</repositories>
```

The examples on this page log with Lombok's `@Slf4j`: add Lombok to your build (on JDK 23+ also as
an annotation processor in `maven-compiler-plugin`), or use any other logger.

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
zequent.edge.asset-vendor=ASSET_VENDOR_DJI

# Platform services (read by EdgeWiring below)
grpc.client.live-data.host=${LIVE_DATA_SERVICE_HOST:localhost}
grpc.client.live-data.port=${LIVE_DATA_SERVICE_PORT:8003}
grpc.client.connector.host=${CONNECTOR_SERVICE_HOST:localhost}
grpc.client.connector.port=${CONNECTOR_SERVICE_PORT:8010}

# gRPC server shares HTTP port
quarkus.grpc.server.use-separate-server=false
```

And the adapter's credentials, as environment variables:

```bash
# Edge credential: the Admin Console, Manage -> Access & Integrations -> Credentials (kind "Edge adapter")
ZQNT_EDGE_TOKEN=<your edge credential>
# The platform's public key, to check the platform's calls into the adapter
ZQNT_PLATFORM_PUBLIC_KEY=<the platform's service public key>
```

See [Configuration](edge-sdk-configuration.md#grpc-client-configuration) for both.

---

## Step 3b: Wire the SDK

The SDK registers no beans of its own. Three small classes connect it to Quarkus: one creates the
services your adapter uses (with the edge credential on every channel), one exposes your adapter to
the platform over gRPC, and one checks that calls into it come from the platform.

```java
package com.example.edge;

import com.zqnt.sdk.edge.application.ProtoJsonMapper;
import com.zqnt.sdk.edge.auth.EdgeAuthConfig;
import com.zqnt.sdk.edge.auth.EdgeCredentialsClientInterceptor;
import com.zqnt.sdk.edge.connector.application.ConnectorService;
import com.zqnt.sdk.edge.connector.application.impl.ConnectorServiceImpl;
import com.zqnt.sdk.edge.livedata.application.DetectionMapper;
import com.zqnt.sdk.edge.livedata.application.LiveDataService;
import com.zqnt.sdk.edge.livedata.application.NotificationMapper;
import com.zqnt.sdk.edge.livedata.application.TelemetryMapper;
import com.zqnt.sdk.edge.livedata.application.impl.LiveDataServiceImpl;
import com.zqnt.utils.connector.proto.ConnectorServiceGrpc;
import com.zqnt.utils.livedata.proto.LiveDataServiceGrpc;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

// The SDK registers no beans of its own: this class creates what the adapter uses.
@ApplicationScoped
public class EdgeWiring {

    @ConfigProperty(name = "grpc.client.live-data.host") String liveDataHost;
    @ConfigProperty(name = "grpc.client.live-data.port") int liveDataPort;
    @ConfigProperty(name = "grpc.client.connector.host") String connectorHost;
    @ConfigProperty(name = "grpc.client.connector.port") int connectorPort;

    // Puts ZQNT_EDGE_TOKEN on every call to the platform
    private final EdgeCredentialsClientInterceptor credentials =
            new EdgeCredentialsClientInterceptor(EdgeAuthConfig.fromEnv().edgeToken());

    @Produces
    @ApplicationScoped
    ProtoJsonMapper protoJsonMapper() {
        return new ProtoJsonMapper();
    }

    @Produces
    @ApplicationScoped
    LiveDataService liveDataService() {
        return new LiveDataServiceImpl(new TelemetryMapper(), new DetectionMapper(), new NotificationMapper(),
                LiveDataServiceGrpc.newStub(channel(liveDataHost, liveDataPort)));
    }

    @Produces
    @ApplicationScoped
    ConnectorService connectorService(ProtoJsonMapper mapper) {
        return new ConnectorServiceImpl(mapper, ConnectorServiceGrpc.newStub(channel(connectorHost, connectorPort)));
    }

    private ManagedChannel channel(String host, int port) {
        return ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .intercept(credentials)
                .build();
    }
}
```

```java
package com.example.edge;

import com.zqnt.sdk.edge.adapter.application.EdgeAdapterService;
import com.zqnt.sdk.edge.adapter.grpc.EdgeAdapterGrpcServiceImpl;
import com.zqnt.sdk.edge.application.ProtoJsonMapper;
import io.quarkus.grpc.GrpcService;

// Exposes your EdgeAdapterService to the platform over gRPC
@GrpcService
public class EdgeGrpcService extends EdgeAdapterGrpcServiceImpl {
    public EdgeGrpcService(EdgeAdapterService adapter, ProtoJsonMapper mapper) {
        super(adapter, mapper);
    }
}
```

```java
package com.example.edge;

import com.zqnt.sdk.edge.auth.EdgeAuthConfig;
import com.zqnt.sdk.edge.auth.PlatformAuthServerInterceptor;
import io.grpc.Metadata;
import io.grpc.ServerCall;
import io.grpc.ServerCallHandler;
import io.grpc.ServerInterceptor;
import io.quarkus.grpc.GlobalInterceptor;
import jakarta.enterprise.context.ApplicationScoped;

// Refuses every call into the adapter that the platform did not sign
@GlobalInterceptor
@ApplicationScoped
public class PlatformAuth implements ServerInterceptor {
    private final ServerInterceptor delegate = new PlatformAuthServerInterceptor(EdgeAuthConfig.fromEnv());

    @Override
    public <Q, R> ServerCall.Listener<Q> interceptCall(ServerCall<Q, R> call, Metadata headers,
                                                       ServerCallHandler<Q, R> next) {
        return delegate.interceptCall(call, headers, next);
    }
}
```

Health probes stay open. Without `ZQNT_PLATFORM_PUBLIC_KEY` every command is refused;
`ZQNT_EDGE_AUTH_DISABLED=true` turns the check off, for a local stack only. Add channels for
Mission Autonomy (`MissionAutonomyServiceImpl`) the same way when you need it.

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
Starting new Quarkus gRPC server (using Vert.x transport)...
my-edge-adapter 1.0.0-SNAPSHOT on JVM (powered by Quarkus 3.x) started in 0.4s. Listening on: http://0.0.0.0:9001
```

Your adapter is now running and ready to receive commands from the platform via gRPC.

---

## Step 8: Test with grpcurl

You can test your adapter endpoint with `grpcurl`. Your adapter refuses calls the platform did not
sign, so for this test run it with `ZQNT_EDGE_AUTH_DISABLED=true` — locally only — and with
`quarkus.grpc.server.enable-reflection-service=true` (on by default only in dev mode), so `grpcurl`
can discover the services:

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
    "timestamp": "2026-01-01T00:00:00Z"
  },
  "coordinate": {
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
          EdgeWiring.java             # SDK services, with the edge credential
          EdgeGrpcService.java        # Exposes the adapter over gRPC
          PlatformAuth.java           # Checks the platform's calls in
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
- [Connector](edge-sdk-connector.md) -- Asset pairing and the Skill Registry
- [Models Reference](../api-reference/edge-sdk-models.md) -- Complete model documentation

For a ready-made DJI deployment, see [DJI Adapter Deployment](edge-sdk-dji-adapter-deployment.md).
