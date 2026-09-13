# Minimal Customer Example

A from-scratch walkthrough: scaffold a new Quarkus project, add the SDK, and have a working
`/drone/takeoff` endpoint running in a container. For an explanation of each piece (CDI injection,
standalone usage, configuration, the other client interfaces), see [QUICKSTART.md](QUICKSTART.md) —
this page assumes you've skimmed it and just want the sequence of steps.

## Step by Step: Your First Project with Zequent Client SDK

### 1. Create New Quarkus Project

```bash
mvn io.quarkus:quarkus-maven-plugin:3.17.4:create \
    -DprojectGroupId=com.example \
    -DprojectArtifactId=drone-app \
    -Dextensions="resteasy-reactive-jackson,arc"

cd drone-app
```

### 2. Add Zequent Client SDK Dependency

Open `pom.xml` and add:

```xml
<dependency>
    <groupId>com.zqnt.sdk</groupId>
    <artifactId>client-java-sdk</artifactId>
    <version>1.3.2</version>
</dependency>
```

### 3. Create .env File

```bash
# .env in project root
REMOTE_CONTROL_SERVICE_HOST=localhost
REMOTE_CONTROL_SERVICE_PORT=8002
LIVE_DATA_SERVICE_HOST=localhost
LIVE_DATA_SERVICE_PORT=8003
```

### 4. Write Your First API

Create `src/main/java/com/example/DroneResource.java`:

```java
package com.example;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.remotecontrol.domains.*;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import java.util.concurrent.CompletableFuture;

@Path("/drone")
@Produces(MediaType.APPLICATION_JSON)
public class DroneResource {

    @Inject
    ZequentClient zequent;  // ← Automatically configured!

    @POST
    @Path("/takeoff")
    public CompletableFuture<TakeoffResponse> takeoff(
            @QueryParam("sn") String sn,
            @QueryParam("lat") float lat,
            @QueryParam("lon") float lon,
            @QueryParam("alt") float alt) {

        var request = TakeoffRequest.builder()
            .sn(sn)
            .latitude(lat)
            .longitude(lon)
            .altitude(alt)
            .build();

        // Automatic Retry, Circuit Breaker, Load Balancing!
        return zequent.remoteControl().takeoff(request);
    }

    @POST
    @Path("/land")
    public CompletableFuture<RemoteControlResponse> land(@QueryParam("sn") String sn) {
        var request = ReturnToHomeRequest.builder()
            .sn(sn)
            .build();

        return zequent.remoteControl().returnToHome(request);
    }
}
```

### 5. Start Your Application Container

```bash
docker run --env-file .env --network host your-registry/drone-app:latest
```

### 6. Test

```bash
# Takeoff
curl -X POST "http://localhost:8080/drone/takeoff?sn=YOUR_DEVICE_SN&lat=47.3769&lon=8.5417&alt=100"

# Land
curl -X POST "http://localhost:8080/drone/land?sn=YOUR_DEVICE_SN"
```

That's the whole integration — no interfaces to implement, no manual gRPC channel setup.

## Switch Environment

Moving this same project from your machine to staging to Kubernetes is a `.env`/deployment-config
change, not a code change — see [QUICKSTART.md — Environment-Specific Configuration](QUICKSTART.md#environment-specific-configuration)
for the Docker Compose and Kubernetes settings.

## Complete Project Structure

```
drone-app/
 pom.xml                      # With Zequent SDK dependency
 .env                         # Service Configuration
 src/
    main/
        java/
           com/example/
               DroneResource.java   # Your API
        resources/
            application.properties   # Optional: Defaults
 docker-compose.yml           # Optional: For Staging
```

## More Examples

### Service with Business Logic

```java
package com.example;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.remotecontrol.domains.*;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

@ApplicationScoped
public class DroneFlightService {

    @Inject
    ZequentClient zequent;

    public boolean executeMission(String sn, List<Waypoint> waypoints) {
        // 1. Takeoff
        var takeoffResponse = zequent.remoteControl().takeoff(
            TakeoffRequest.builder()
                .sn(sn)
                .latitude(waypoints.get(0).getLat())
                .longitude(waypoints.get(0).getLon())
                .altitude(100.0f)
                .build()
        ).join();

        if (!takeoffResponse.isSuccess()) {
            return false;
        }

        // 2. Fly waypoints
        for (Waypoint wp : waypoints) {
            var goToResponse = zequent.remoteControl().goTo(
                GoToRequest.builder()
                    .sn(sn)
                    .latitude(wp.getLat())
                    .longitude(wp.getLon())
                    .altitude(wp.getAlt())
                    .build()
            ).join();

            if (!goToResponse.isSuccess()) {
                return false;
            }
        }

        // 3. Return to home
        var rthResponse = zequent.remoteControl().returnToHome(
            ReturnToHomeRequest.builder()
                .sn(sn)
                .build()
        ).join();

        return rthResponse.isSuccess();
    }
}
```

### WebSocket for Live Telemetry

```java
package com.example;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.livedata.domains.StreamTelemetryRequest;
import com.zqnt.sdk.client.livedata.domains.StreamTelemetryResponse;
import jakarta.inject.Inject;
import jakarta.websocket.*;
import jakarta.websocket.server.ServerEndpoint;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@ServerEndpoint("/ws/telemetry/{sn}")
public class TelemetryWebSocket {

    @Inject
    ZequentClient zequent;

    @OnOpen
    public void onOpen(Session session, @PathParam("sn") String sn) {
        log.info("Client connected: {}", sn);

        var request = StreamTelemetryRequest.builder()
            .sn(sn)
            .build();

        // Stream telemetry to WebSocket client
        zequent.liveData().streamTelemetryData(
            request,
            telemetry -> {
                try {
                    session.getBasicRemote().sendText(telemetry.toString());
                } catch (Exception e) {
                    log.error("Failed to send telemetry", e);
                }
            },
            error -> log.error("Telemetry stream error", error)
        );
    }
}
```

## Summary

That's the full loop: scaffold, add the dependency, inject `ZequentClient`, and it just works — see
[QUICKSTART.md — Built-in Features](QUICKSTART.md#built-in-features) for what the SDK is doing for
you under the hood (retry, circuit breaker, load balancing, service discovery).

For flying a waypoint mission instead of one-off commands, see [Waypoint Missions](WAYPOINT_MISSIONS.md).

## Support

- Complete docs: [CONFIGURATION.md](CONFIGURATION.md)
- Quick start: [QUICKSTART.md](QUICKSTART.md)
- Documentation: [../README.md](../README.md)
