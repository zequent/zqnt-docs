# Spring Boot Integration

The **simplest** way to use ZequentClient in Spring Boot.

Uses `ZequentClientProducer` internally for consistent bean creation.

---

## Solution 1: Ultra Simple (Only Defaults)

### Step 1: Maven Dependency

```xml
<dependency>
    <groupId>com.zqnt.sdk</groupId>
    <artifactId>client-java-sdk</artifactId>
    <version>1.3.2</version>
</dependency>
```

### Step 2: Write One Bean Method

**`src/main/java/com/yourcompany/config/ZequentConfig.java`**

```java
package com.yourcompany.config;

import com.zqnt.sdk.client.ZequentClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ZequentConfig {

    @Bean
    public ZequentClient zequentClient() {
        // Uses all defaults: localhost:8002, 8004, 8003
        return ZequentClient.builder()
                .remoteControl().done()
                .missionAutonomy().done()
                .liveData().done()
                .build();
    }
}
```

### Step 3: Use Constructor Injection

```java
package com.yourcompany.service;

import com.zqnt.sdk.client.ZequentClient;
import com.zqnt.sdk.client.livedata.domains.StreamTelemetryRequest;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
@RequiredArgsConstructor
public class LiveDataService {

    private final ZequentClient zequentClient;  // ← Automatically injected!

    public void streamTelemetry(StreamTelemetryRequest request) {
        zequentClient.liveData().streamTelemetryData(request, telemetry -> log.info("{}", telemetry));
    }
}
```

That's the whole integration.

---

## Solution 2: With Properties (Optional)

If you want to use properties from `application.properties`:

### Bean with @Value

```java
package com.yourcompany.config;

import com.zqnt.sdk.client.ZequentClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ZequentConfig {

    @Bean
    public ZequentClient zequentClient(
            @Value("${zequent.remote-control.host:localhost}") String rcHost,
            @Value("${zequent.remote-control.port:8002}") int rcPort,
            @Value("${zequent.mission-autonomy.host:localhost}") String maHost,
            @Value("${zequent.mission-autonomy.port:8004}") int maPort,
            @Value("${zequent.live-data.host:localhost}") String ldHost,
            @Value("${zequent.live-data.port:8003}") int ldPort) {

        return ZequentClient.builder()
                .remoteControl()
                    .host(rcHost)
                    .port(rcPort)
                    .done()
                .missionAutonomy()
                    .host(maHost)
                    .port(maPort)
                    .done()
                .liveData()
                    .host(ldHost)
                    .port(ldPort)
                    .done()
                .build();
    }
}
```

### application.properties

```properties
# Only what you want to change - rest uses defaults
zequent.remote-control.host=prod-server.com
zequent.live-data.port=9999
```

---

## Recommendation

For most customers, **Solution 1** or **Solution 2** is enough — `ZequentClient.builder()` gives you the same resilience, load balancing, and connection management as the Quarkus/CDI path, just wired through a Spring `@Bean` instead of `@Inject`.

## Purpose of Components

### `ZequentClientProducer` (Quarkus only)
- Used automatically by Quarkus CDI.
- Reads configuration via `@ConfigMapping`.
- Not relevant outside Quarkus — Spring Boot applications don't use it.

### `ZequentClient.builder()` (Spring Boot & standalone)
- For manual bean creation.
- Uses sensible defaults (`localhost:8002/8004/8003`).
- Fully configurable per service.

Both paths produce the same `ZequentClient` and the same runtime behavior — the builder is simply the non-CDI equivalent of what the Quarkus producer does automatically.

---

## Usage

Usage is identical regardless of which solution wires up the bean:

```java
@Service
@RequiredArgsConstructor
public class MyService {
    private final ZequentClient zequentClient;

    public void doSomething() {
        zequentClient.liveData().streamTelemetryData(...);
        zequentClient.remoteControl().takeoff(...);
    }
}
``` 
