# Edge SDK -- Edge Adapter Service

The `EdgeAdapterService` interface is the core contract of the Edge SDK. Every edge adapter must provide a CDI bean that implements this interface. The SDK ships with a default implementation (`EdgeAdapterServiceImpl`) whose methods all return `NOT_IMPLEMENTED`, so you only need to override the commands that your particular hardware supports.

Full method-by-method reference: [Edge Adapter API Reference](../api-reference/edge-sdk-adapter-reference.md).

## Table of Contents

- [How It Works](#how-it-works)
- [Creating an Adapter](#creating-an-adapter)
- [Custom Commands](#custom-commands)
  - [Command ID naming convention](#command-id-naming-convention)
- [gRPC Layer](#grpc-layer)
- [Best Practices](#best-practices)

---

## How It Works

When your Quarkus application starts, the SDK registers a gRPC service (`EdgeAdapterGrpcServiceImpl`) that receives commands from the platform and delegates them to your `EdgeAdapterService` bean. The flow is:

```
Platform Services  --(gRPC)-->  EdgeAdapterGrpcServiceImpl  --(delegates)-->  Your EdgeAdapterService implementation
```

Every command method returns `CompletableFuture<CommandResult>`, which means your implementation can be fully asynchronous. The gRPC layer wraps it in a Mutiny `Uni` automatically.

---

## Creating an Adapter

### Step 1: Implement the Interface

Create a CDI bean that implements `EdgeAdapterService`. The `@ApplicationScoped` annotation ensures there is a single instance across the application lifecycle.

```java
package com.example.edge;

import com.zqnt.sdk.edge.adapter.application.EdgeAdapterService;
import com.zqnt.sdk.edge.adapter.domains.*;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.concurrent.CompletableFuture;

@ApplicationScoped
public class MyDeviceAdapter implements EdgeAdapterService {

    @Override
    public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
        // Call your device SDK/API here
        boolean success = myDevice.initiateTakeoff(
            request.getCoordinates().getLatitude(),
            request.getCoordinates().getLongitude(),
            request.getCoordinates().getAltitude()
        );

        if (success) {
            return CompletableFuture.completedFuture(
                CommandResult.success("Takeoff initiated", request.getTid(), request.getSn())
            );
        } else {
            return CompletableFuture.completedFuture(
                CommandResult.error("Takeoff failed: device busy", request.getSn())
            );
        }
    }
}
```

### Step 2: Asynchronous Implementation

If your device SDK provides asynchronous or callback-based APIs, use `CompletableFuture` accordingly:

```java
@Override
public CompletableFuture<CommandResult> takeOff(TakeOffRequest request) {
    CompletableFuture<CommandResult> future = new CompletableFuture<>();

    myDevice.takeoffAsync(request.getCoordinates(), new DeviceCallback() {
        @Override
        public void onSuccess() {
            future.complete(CommandResult.success("Takeoff complete", request.getSn()));
        }

        @Override
        public void onError(String errorMsg) {
            future.complete(CommandResult.error(errorMsg, request.getSn()));
        }
    });

    return future;
}
```

### Step 3: Override Only What You Support

Any method that you do not override will automatically return a `NOT_IMPLEMENTED` result to the caller. This is by design -- a dock adapter may support `openCover` and `startCharging` but not `takeOff` (which is a drone-level command), and that is perfectly fine.

See the [API Reference](../api-reference/edge-sdk-adapter-reference.md) for the full command surface, grouped
by area (Flight Control, Dock Operations, Camera and Gimbal, Manual Control, Live Streaming, Debug
and Maintenance, Task Execution, Capability Reporting), plus `CommandResult`, the default
convenience-method overloads, and the error-code mapping.

---

## Custom Commands

For a command that doesn't map to a standard `EdgeAdapterService` method, override `sendCustomCommand`:

```java
@Override
public CompletableFuture<CommandResult> sendCustomCommand(String sn, String componentId,
        String commandType, Map<String, Object> params) {
    if ("mission.waypoint.execute".equals(commandType)) {
        String executionId = deviceApi.startWaypointMission(params);
        return CompletableFuture.completedFuture(
            CommandResult.success("Waypoint mission started", executionId, sn)
        );
    }
    return CompletableFuture.completedFuture(CommandResult.notImplemented("Unknown command", sn));
}
```

This is the path MAVLink and the simulator use for waypoint missions instead of the Task Execution
methods (`prepareTask`/`startTask`) — waypoints and configuration arrive inline in `params`, so no
task lookup is needed. See [Task Execution](../api-reference/edge-sdk-adapter-reference.md#task-execution) for
the alternative, task-ID-based path DJI and SAPIENT use, and
[Waypoint Missions](../client-sdk/WAYPOINT_MISSIONS.md#which-path-does-your-adapter-use) for the
full per-adapter picture. Both approaches are valid; leaving a method unimplemented returns
`NOT_IMPLEMENTED`, which callers handle. Whichever you choose, document it, because the two are not
interchangeable from a customer application's point of view.

### Command ID naming convention

Every built-in command maps to a well-known, vendor-neutral `command_id` string. Custom commands should follow the same convention:

- **Vendor-neutral** — never encode a vendor name (`dji.takeoff` is wrong); the same id should be implementable by any adapter.
- **`domain.action`** for a single atomic command — a domain (`flight`, `navigation`, `dock`, `asset`, `camera`, `gimbal`, `stream`) and a snake_case action (`return_to_home`, `go_to`, `change_lens`).
- **`domain.subtype.action`** only when a domain genuinely has distinct execution variants, e.g. `mission.waypoint.execute`.
- **`custom.` prefix** for tenant/user-defined commands, so they never collide with a future built-in of the same name.
- Commands under `flight.`, `navigation.`, `dock.`, `mission.`, and `asset.reboot` are treated as high-risk by the platform's execution safety checks (e.g. may require human approval) — keep new movement-capable commands under one of those prefixes rather than introducing an unrecognized domain.

---

## gRPC Layer

The `EdgeAdapterGrpcServiceImpl` class is registered as a `@GrpcService`. It:

1. Receives incoming gRPC requests from the platform (Remote Control Service, Client SDK).
2. Maps Proto request messages to SDK model POJOs using `ProtoJsonMapper`.
3. Delegates to your `EdgeAdapterService` implementation.
4. Converts `CommandResult` back to an `EdgeResponse` Proto message.
5. Handles errors with proper gRPC error codes and `GlobalErrorMessage`.

You typically do not need to interact with this class directly. It is wired automatically by the CDI container and the Quarkus gRPC extension.

---

## Best Practices

1. **Override selectively.** Only implement the commands your hardware actually supports. The default `NOT_IMPLEMENTED` response gives callers a clear signal that a given command is not available.

2. **Use CompletableFuture properly.** Do not block inside your adapter methods. If your device SDK is blocking, wrap the call in `CompletableFuture.supplyAsync(...)`.

3. **Report capabilities.** Override `getCapabilities` to give the platform and its users an accurate view of what your device can do at any given moment.

4. **Use transaction IDs.** Pass the `tid` from the request through to the `CommandResult` so that command execution can be correlated across the system.

5. **Handle timeouts.** If your device takes time to respond, use `CompletableFuture.orTimeout(...)` or custom timeout logic so that callers are not left waiting indefinitely.

6. **Log meaningfully.** The gRPC layer already logs incoming commands. Focus your adapter logs on device-level events, errors, and state changes.
