# Zequent Client SDK (Java) — Commands (3.0 preview)

> **Preview.** This page describes the Zequent **3.0** command API of the Java Client SDK. 3.0 is not
> released yet: the API below is available as a preview build and needs a platform that runs 3.0.
> Names may still change before the release. For the current 2.0 line, see
> [Remote Control](REMOTE_CONTROL.md) and the [Java Quickstart](QUICKSTART.md).

On a 3.0 platform every asset is commanded the same way: ask the asset what it can do, then run a
command by its dotted **command id** (`flight.takeoff`, `navigation.go_to`, `dock.open_cover`, …)
with a JSON object of params. There is no typed method per command, so a command a new adapter adds
works without a new SDK release.

Python: [Commands (Python)](COMMANDS_PYTHON-3.0.md) · Go: [Commands (Go)](COMMANDS_GO-3.0.md) ·
Upgrading an integration: [Upgrading to 3.0](../concepts/upgrading-to-3.0.md).

## Step 1: Add the preview dependency

The 3.0 preview is published to GitHub Packages as a `-SNAPSHOT` build of the Java Client SDK. Every
push to the 3.0 branch publishes a new preview version; at the time of writing it is:

```xml
<dependency>
    <groupId>com.zqnt.sdk</groupId>
    <artifactId>client-java-sdk</artifactId>
    <!-- preview, not a release -->
    <version>1.0.0-PR11-fe5e92b-SNAPSHOT</version>
</dependency>
```

The SDK's own repository and the `zqnt-utils-java` package it depends on (a preview build as well)
are both on GitHub Packages, so your `~/.m2/settings.xml` needs a token with package read access for:

```xml
<repository>
    <id>zqnt-client-sdk-repo</id>
    <url>https://maven.pkg.github.com/Zequent/zqnt-client-sdk-java</url>
</repository>
<repository>
    <id>zqnt-utils-repo</id>
    <url>https://maven.pkg.github.com/Zequent/zqnt-utils-java</url>
</repository>
```

Move to the released `3.0.0` version once it is out; do not run a preview in production.

Configuration is unchanged from 2.0: the same `.env` / `application.properties` and the same
`ZQNT_CLIENT_TOKEN` client credential (see [Client SDK Configuration](CONFIGURATION.md)).
`zequent.commands()` uses the remote-control service connection (`REMOTE_CONTROL_SERVICE_HOST` /
`_PORT`), credential and interceptors the client already has — nothing else to set up.

## Step 2: Discover what an asset can do

`zequent` below is your `ZequentClient`, injected or built as in the [Java Quickstart](QUICKSTART.md).

```java
import com.zqnt.protos.capability.v3.CapabilitySet;
import com.zqnt.sdk.client.commands.application.Commands;

Commands commands = zequent.commands();

CapabilitySet capabilities = commands.listCapabilities("DOCK-1").join();
capabilities.getCapabilitiesList().forEach(c ->
        System.out.println(c.getCommandId() + " " + c.getSafety().getRisk() + " " + c.getInputSchema()));
```

`listCapabilities(assetSn)` returns the asset's `CapabilitySet` (`com.zqnt.protos.capability.v3`):
one `Capability` per command, with

- `getCommandId()` — the dotted id you execute, and `getDisplayName()` / `getDescription()`;
- `getInputSchema()` / `getOutputSchema()` — JSON Schemas of the params and of the result;
- `getSafety().getRisk()` — `COMMAND_RISK_OBSERVE`, `_ADJUST`, `_MOVE` or `_CRITICAL`
  (a command without a rating counts as CRITICAL);
- `getState()` — whether the command is available right now, with `getUnavailableReason()`;
- `getTarget()` — which part of the asset executes it (the asset, a sub-asset, a payload, a component);
- `getCompletion()` — `COMPLETION_MODE_ON_REPLY` (the reply is the outcome) or
  `COMPLETION_MODE_ASYNCHRONOUS` (the outcome follows as an event — takeoff, go-to, return home);
- `getErrorsList()` / `getEventsList()` — the error codes and events the command declares.

The set's `getSnapshotState()` is `SNAPSHOT_STATE_STALE` when the platform could not reach the
asset's adapter and answers with the last snapshot it kept.

The capability's input schema is authoritative for an asset. The built-in commands every Zequent
adapter shares are listed in [Upgrading to 3.0](../concepts/upgrading-to-3.0.md#typed-call--command-id).

## Step 3: Execute a command by id

```java
import com.zqnt.protos.capability.v3.CommandResult;
import java.util.Map;

CommandResult goTo = commands.executeCommand("DOCK-1", "navigation.go_to",
        Map.of("latitude", 47.7760, "longitude", 9.2671, "altitude", 60)).join();

System.out.println(goTo.getState() + " " + goTo.getCommandExecutionId());
```

`executeCommand(assetSn, commandId, params)` returns a `CompletableFuture<CommandResult>`:

- a command that finishes with the reply comes back in its final state (`COMMAND_STATE_SUCCEEDED`
  with `getResult()`, or `COMMAND_STATE_FAILED` with `getError()`);
- a command that takes time comes back `COMMAND_STATE_ACCEPTED` or `COMMAND_STATE_RUNNING` with a
  `getCommandExecutionId()`; its outcome follows as events (Step 5) — or use `executeAndWait` (Step 4).

Params are a `Map` turned into a JSON object. **Leave a param out when you have no value**: the
platform reads a missing coordinate as "not given", while `0` means latitude 0 / longitude 0 /
ground level. A `null` value in the map is left out for you. `navigation.go_to`'s `altitude` is
metres above the **takeoff point** (left out = 40 m).

### Options

`executeCommand(CommandRequest)` adds options:

```java
import com.zqnt.sdk.client.commands.domains.CommandRequest;
import java.time.Duration;

commands.executeCommand(CommandRequest.builder()
        .assetSn("DOCK-1")
        .commandId("flight.return_to_home")
        .param("altitude", 80)
        .timeout(Duration.ofMinutes(10))
        .reason("battery low")
        .idempotencyKey(myRequestId)
        .build());
```

| `CommandRequest` field | Meaning |
| --- | --- |
| `assetSn` / `assetId` | The asset, by serial number or by platform id (one of the two) |
| `commandId` | The dotted command id |
| `param(name, value)` / `params(map)` | The command's params |
| `target` | A `com.zqnt.protos.capability.v3.Target` — a sub-asset, payload or component instead of the asset itself |
| `timeout` | Replaces the capability's own default timeout |
| `reason` | A human reason; **required** for `COMMAND_RISK_CRITICAL` commands |
| `noFlyZoneOverride` | Fly through no-fly zones instead of around them; honoured for an organization admin or a system admin only, anybody else's command is refused |
| `idempotencyKey` | The same key sent twice runs the command once. Unset, the SDK uses a fresh key per call, kept across its own retries |

## Step 4: Execute and wait for the outcome

```java
CommandResult takeoff = commands.executeAndWait("DOCK-1", "flight.takeoff",
        Map.of("latitude", 47.7752, "longitude", 9.2658, "altitude", 40), Duration.ofMinutes(5)).join();
```

`executeAndWait(assetSn, commandId, params, wait)` (or `executeAndWait(CommandRequest, wait)`) runs
the command and completes with its outcome:

- It opens a watch on the asset's command events **before** it sends the command, so a run that
  finishes right away is not missed.
- It follows **only the execution id** this call started; events of other runs on the same asset
  are ignored.
- **Only `COMMAND_STATE_SUCCEEDED` is success.** The future completes with that result; a run that
  ends `REJECTED`, `FAILED`, `CANCELLED` or `TIMED_OUT` fails the future with a `CommandException`
  whose `getResult()` is the final result.
- `wait` bounds the wait (`null` waits without a limit). When it elapses the future fails with a
  `CommandException` of category `ERROR_CATEGORY_TIMEOUT` and no result — **the command keeps
  running** on the platform; `cancelCommand` stops it.
- Completing or cancelling the future closes the watch. If the watch ends before the run does, the
  future fails with category `ERROR_CATEGORY_SERVICE` (retryable).
- The asset must be named by `assetSn` here (`assetId` alone is not enough to watch it).

## Step 5: Watch command events

```java
import com.zqnt.sdk.client.commands.domains.CommandWatch;
import java.util.concurrent.TimeUnit;

try (CommandWatch watch = commands.watchCommand(goTo.getCommandExecutionId(),
        event -> System.out.println(event.getState() + " " + event.getProgress() + " " + event.getMessage()))) {
    watch.done().get(5, TimeUnit.MINUTES);
}
```

- `watchCommand(commandExecutionId, onEvent)` — events of one run **from now on**, ending with its
  terminal event.
- `watchAsset(assetSn, onEvent)` — events of every command on the asset from now on, until you close
  the watch.

Both return a `CommandWatch` (`AutoCloseable`): `done()` completes when the platform ends the watch
or `close()` is called, and fails with a `CommandException` when the platform refuses or drops it.
Each `CommandEvent` carries `getCommandExecutionId()`, `getCommandId()`, `getState()`,
`getProgress()` (0..1, when the adapter can tell — check `hasProgress()`), `getRemaining()`,
`getMessage()`, and on the final event `getResult()` / `getError()`.

A watch opened after `executeCommand` returns can miss the end of a fast run. To wait for a run's
outcome, use `executeAndWait`.

## Step 6: Cancel a run

```java
CommandResult cancelled = commands.cancelCommand(goTo.getCommandExecutionId(), "operator abort").join();
```

`cancelCommand(commandExecutionId, reason)` returns the run's `CommandResult`; a cancel the platform
rejects fails the future with a `CommandException`.

## Results and errors

`Structs.toMap(result.getResult())` (`com.zqnt.sdk.client.commands.domains.Structs`) turns a result
or event payload into a plain `Map`. Numbers come back as `Double`, as in any JSON object.

Every future fails with a `CommandException` (`com.zqnt.sdk.client.commands.domains`) when:

- the platform refused the call — `getStatus()` is the gRPC status (`PERMISSION_DENIED`,
  `INVALID_ARGUMENT`, `UNAVAILABLE`, …);
- the command was `REJECTED` before it started (unknown id, params, precondition, permission) —
  `getResult()` is the rejected result, `getCode()` its stable code such as `command.invalid_params`;
- from `executeAndWait` only: the run ended `FAILED`, `CANCELLED` or `TIMED_OUT`.

`getCategory()` (a `com.zqnt.protos.common.v3.ErrorCategory`) says whose problem it is —
`ERROR_CATEGORY_INVALID_ARGUMENT`, `_PERMISSION_DENIED`, `_PRECONDITION_FAILED`, `_ASSET`,
`_SERVICE`, `_TIMEOUT`, `_NOT_FOUND`; `isRetryable()` whether the same command may succeed if sent
again.

From `executeCommand`, a command that **started and then failed** is not an exception: the future
completes with a `CommandResult` in state `COMMAND_STATE_FAILED`, with `getError()`
(`getCategory()`, `getCode()` such as `flight.not_airborne`, `getMessage()`, `getRetryable()`).

```java
commands.executeAndWait("DOCK-1", "dock.open_cover", Map.of(), Duration.ofMinutes(2))
        .exceptionally(failure -> {
            CommandException error = CommandException.from(failure);
            System.out.println(error.getCategory() + " " + error.getCode() + " " + error.getMessage());
            return null;
        });
```

## Manual control (stick input)

The 3.0 contract has a manual-control stream (`StreamManualControl`), but the Java Client SDK does
not wrap it yet — **coming**. Keep using `zequent.remoteControl().startManualControlInput(sn, assetId)`
for stick input; it is not deprecated. Entering and leaving manual control are commands on 3.0:
`flight.manual.enter` and `flight.manual.exit`.

## Against a 2.x platform

`zequent.commands()` needs a 3.0 platform. A 2.x platform does not serve the 3.0 command service:
the calls fail with a `CommandException` whose `getStatus()` is `UNIMPLEMENTED` (category
`ERROR_CATEGORY_NOT_FOUND`). The SDK does **not** switch to the 2.x calls on its own. Against 2.x,
keep using the typed `zequent.remoteControl()` methods, which still work in this SDK and are
marked `@Deprecated(since = "3.0")`. See
[Upgrading to 3.0](../concepts/upgrading-to-3.0.md#running-against-2x-platforms).
