# Flying a Waypoint Mission

A waypoint flight is **one command**, `mission.waypoint.execute`, with the waypoints and settings
inline. This page covers how to send it, the configuration model, pausing and resuming, and how to
track progress.

> **Upgrading from 1.3?** The task-based path (`createTask` + `startTask`) is gone: the platform no
> longer calls an adapter's task methods. See
> [Upgrading from 1.3](../concepts/migration-guide.md#task-based-execution-is-gone).

## Which adapters support it

| Adapter (`2.0.0`) | `mission.waypoint.execute` | Pause / resume |
| --- | --- | --- |
| **DJI** | Yes | `mission.pause` / `mission.resume` |
| **MAVLink** | Yes | `mission.pause` / `mission.resume` |
| **Simulator** | Yes | `mission.pause` / `mission.resume` |
| **SAPIENT**, **Betaflight**, **RNS** | No | — |

Ask an asset before you send it — `getCapabilities(sn)` lists the commands it advertises (see
[Checking what an asset supports](#checking-what-an-asset-supports)).

## Building the configuration

The command's parameters are a `WaypointTaskConfig` (see [Configuration](#configuration)):

```java
import com.zqnt.utils.JsonUtils;
import com.zqnt.utils.missionautonomy.domains.WaypointDTO;
import com.zqnt.utils.missionautonomy.domains.config.WaypointTaskConfig;

WaypointTaskConfig config = WaypointTaskConfig.builder()
        .waypoints(List.of(
            WaypointDTO.builder()
                .latitude(52.52000045776367).longitude(13.404999732971191)
                .altitude(30.0f).speed(5.0f).wpOrder(0).build(),
            WaypointDTO.builder()
                .latitude(52.541083304641944).longitude(13.40373026335974)
                .altitude(30.0f).speed(5.0f).wpOrder(1).build()))
        .globalSpeed(5.0f)
        .globalHeight(50.0f)
        .build();

config.validate();   // same rules the adapter applies — fail fast client-side
```

### Use `JsonUtils.getMapper()` for the conversion

`JsonUtils`'s mapper is configured with `Include.NON_NULL`, so fields you did not set are omitted
and the class defaults apply adapter-side.

A **default-configured** `ObjectMapper` serialises nulls instead. Those nulls survive the whole
way — the SDK encodes them as protobuf `NullValue`, and the adapter then deserialises
`"globalSpeed": null` **over** the field default, wiping it. You would silently lose `globalSpeed`,
`waylineFinishAction`, `rcLostActionEnum` and the other safety defaults.

If you use your own mapper, set `setSerializationInclusion(JsonInclude.Include.NON_NULL)`.
Unknown properties are ignored adapter-side, so the extra `configType` / `taskType` properties
Jackson emits are harmless.

## Sending it

### Recommended — as a Skill execution

Run the command through the execution engine. The platform then tracks it like any other execution:
it has an id, a status and progress, and it can be cancelled.

```java
import com.google.protobuf.Struct;
import com.google.protobuf.util.JsonFormat;
import com.zqnt.sdk.client.missionautonomy.capabilities.SkillExecutionCommand;

Struct.Builder parameters = Struct.newBuilder();
JsonFormat.parser().merge(JsonUtils.getMapper().writeValueAsString(config), parameters);

var execution = client.missionAutonomy().executeSkill(
        SkillExecutionCommand.simple("SIM-DRONE-001",   // the Asset SN — see "Which SN" below
                "mission.waypoint.execute",
                null,                                    // target: null = the asset itself
                parameters.build(),
                null))                                   // idempotency key
        .join();

log.info("execution {} is {}", execution.getId(), execution.getStatus());
```

`getSkillExecution(id)` reads it back, and `cancelSkillExecution(...)` stops it. A waypoint route can
equally be one step of a Skill in an Application — see
[Applications & Skills](../concepts/applications-and-skills.md). Python
(`execute_simple(sn, "mission.waypoint.execute", params)`) and Go (`ma.ExecuteSimple(...)`) have the
same call.

### Directly — as a custom command

The same command, sent straight to the device. The platform does not track it as an execution:

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.zqnt.sdk.client.remotecontrol.domains.CustomCommandRequest;

Map<String, Object> params = JsonUtils.getMapper()
        .convertValue(config, new TypeReference<Map<String, Object>>() {});

var response = client.remoteControl().sendCustomCommand(
        CustomCommandRequest.builder()
                .sn("SIM-DRONE-001")
                .commandType("mission.waypoint.execute")
                .params(params)
                .build())
        .join();
if (!response.isSuccess()) {
    log.warn("Rejected: {}", response.getError().getErrorMessage());
}
```

Only `sn` and `commandType` are required on the request; `tid` is generated when omitted.

## Pause and resume

Sent as their own commands, with no params — directly, or as Skill executions like the route itself:

```java
client.remoteControl().sendCustomCommand(
    CustomCommandRequest.builder().sn(sn).commandType("mission.pause").build());

client.remoteControl().sendCustomCommand(
    CustomCommandRequest.builder().sn(sn).commandType("mission.resume").build());
```

Pause holds the aircraft in place; resume continues the same route. On the simulator both are
idempotent — pausing an already-paused mission returns success — and both are **rejected when no
mission is active**, which includes the window after the final waypoint when the aircraft has
already begun its automatic return home.

## Configuration

`mission.waypoint.execute` takes a `WaypointTaskConfig`. Per-waypoint entries (`WaypointDTO`) go in the required
`waypoints` list:

| Field | Type | Notes |
| --- | --- | --- |
| `latitude` | `Double` | **required** |
| `longitude` | `Double` | **required** |
| `altitude` | `Float` | metres |
| `speed` | `Float` | m/s for this leg |
| `flyThrough` | `Boolean` | pass through instead of stopping |
| `vehicleAction` | enum | `VEHICLE_ACTION_NONE`, `VEHICLE_ACTION_TAKEOFF`, `VEHICLE_ACTION_LAND` |
| `wpOrder` | `Integer` | execution order; array order is used when absent |
| `gimbalPitch` | `Integer` | degrees, for this leg |

Mission-level fields, all optional, defaults shown:

| Field | Default | Values / notes |
| --- | --- | --- |
| `name` | — | free text |
| `globalSpeed` | `5.0` | must be > 0 and ≤ 15 |
| `globalTransitionSpeed` | `8.0` | speed between legs |
| `globalHeight` | `50.0` | 0–500 |
| `gimbalPitchMode` | `WGP_MODE_LOOK_DOWN` | `WGP_MODE_MANUAL`, `WGP_MODE_POINT_SETTINGS`, `WGP_MODE_LOOK_DOWN` |
| `globalGimbalPitch` | `-45` | −90 to **+30** |
| `payloadImagingType` | — | free text |
| `flyToWaylineMode` | `FTW_MODE_POINT_TO_POINT` | `FTW_MODE_SAFELY`, `FTW_MODE_POINT_TO_POINT` |
| `waylineType` | `WT_WAYPOINT` | `WT_WAYPOINT`, `WT_MAPPING_2D`, `WT_MAPPING_3D`, `WT_MAPPING_STRIP` |
| `waylineFinishAction` | `WF_ACTION_GO_HOME` | `WF_ACTION_GO_HOME`, `WF_ACTION_NO_ACTION`, `WF_ACTION_AUTO_LANDING`, `WF_ACTION_GOTO_FIRST_WAYPOINT`, `WF_ACTION_STOP` |
| `waylineTurnMode` | `WT_MODE_TO_POINT_AND_PASS_WITH_CONTINUITY_CURVATURE` | also `WT_MODE_COORDINATE_TURN`, `WT_MODE_TO_POINT_AND_STOP_WITH_DISCONTINUITY_CURVATURE`, `WT_MODE_TO_POINT_AND_STOP_WITH_CONTINUITY_CURVATURE` |
| `useStraightLine` | `true` | |
| `waylinePrecisionType` | `PRECISION_GPS` | `PRECISION_GPS`, `PRECISION_RTK` |
| `takeOffSecurityHeight` | `10.0` | 2–1500 |
| `exitWaylineWhenRcLostEnum` | `EWWRL_EXECUTE_RC_LOST_ACTION` | `EWWRL_CONTINUE`, `EWWRL_EXECUTE_RC_LOST_ACTION` |
| `rcLostActionEnum` | `RC_LOST_ACTION_RETURN_HOME` | `RC_LOST_ACTION_HOVER`, `RC_LOST_ACTION_LAND`, `RC_LOST_ACTION_RETURN_HOME` |
| `outOfControlAction` | `OOC_RETURN_TO_HOME` | `OOC_RETURN_TO_HOME`, `OOC_HOVERING`, `OOC_LANDING` |
| `rthAltitude` | — | metres |
| `rthMode` | — | `RTH_MODE_OPTIMAL`, `RTH_MODE_PRESET` |
| `rthSpeed` | — | m/s |

`externalTaskId`, `fileUrl`, `fileMd5`, `flightAreaFileUrl` and `flightAreaChecksum` are populated
by the adapter. Do not set them.

**Fidelity is vendor-specific.** The mission-level fields are DJI/KMZ concepts. DJI honours them
in full. MAVLink applies the per-waypoint fields and ignores most mission-level ones. The simulator
reads only `latitude`, `longitude`, `altitude`, `speed` and `wpOrder`; everything else is accepted
and ignored.

### Validation

`config.validate()` runs the same checks the adapter does:

- at least **2** waypoints
- `globalSpeed` in (0, 15] m/s
- `globalHeight` 0–500 m
- `globalGimbalPitch` −90 to +30 degrees
- `takeOffSecurityHeight` 2–1500 m

### Which SN

`sn` is the **Asset** SN — the entity registered on the platform:

- standalone drone registered as an Asset → the drone's SN
- drone docked in a station → the **dock's** SN (the dock is the Asset, the drone its SubAsset)

See [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md).

## Tracking progress

**As a Skill execution**, the execution itself carries the state: `status` moves through
`SKILL_EXECUTION_STATUS_RUNNING` to `SUCCEEDED`, `FAILED` or `CANCELLED`, with `progress` alongside.
Poll `getSkillExecution(id)`, or watch the Skill execution events on the notification stream.

**Underneath**, the adapter reports the command's progress as **command execution events**, on the
notification stream for either way of sending it:

```java
var notifyRequest = new StreamNotificationRequest();
notifyRequest.setSn("DOCK-SN");

client.liveData().streamNotifications(notifyRequest, n -> {
    var event = n.getCommandExecutionEvent();
    if (event == null) return;

    log.info("{} {} {}", event.getCommandId(), event.getStatus(), event.getProgress());
});
```

`CommandExecutionEvent` carries `commandId`, `status`, `progress`, `message` and `output`. `status`
is `COMMAND_EXECUTION_STATUS_RUNNING` while it flies, then exactly one of `SUCCEEDED`, `FAILED` or
`CANCELLED`.

| Adapter | Progress reporting |
| --- | --- |
| **DJI** | Derived from the dock's own `flighttask_progress` |
| **MAVLink** | `RUNNING` as waypoints are reached, then `SUCCEEDED`; `FAILED` on failure |
| **Simulator** | `RUNNING` with progress as it flies, then one terminal event |

> **`SUCCEEDED` means every waypoint was reached, not that the aircraft has landed.** With the
> default `WF_ACTION_GO_HOME` finish action it is normally still flying home when that event
> arrives. Watch telemetry if you need the actual landing.

## Checking what an asset supports

```java
client.remoteControl().getCapabilities(sn)
```

reports the commands an asset advertises. Note that an adapter's advertised input schema may be
deliberately permissive and list fewer properties than it actually accepts — the tables above are
authoritative for `WaypointTaskConfig`.

## See also

- [Remote Control](REMOTE_CONTROL.md) — the rest of the direct command surface
- [Applications & Skills](../concepts/applications-and-skills.md) — making a route one step of a Skill
- [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) — which SN to address
