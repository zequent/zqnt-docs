# Flying a Waypoint Mission

On the 1.3.x line there are **two ways** a waypoint flight reaches a device, and which one applies
depends on the adapter your asset runs. This page covers both, the shared configuration model, and
how to track progress.

> **Read the table before you write code.** The two paths are not interchangeable. Sending the
> wrong one gets you either `startTask is not implemented for this asset` or a "command is not
> registered" error, and neither message points at the real cause.

## Which path does your adapter use?

| Adapter (released 1.3.x) | Execution path | Pause / resume |
| --- | --- | --- |
| **DJI** `1.3.0` | **Task-based** — `createTask` + `startTask` | `pauseTask` / `resumeTask` |
| **MAVLink** `1.3.0` | **Command-based** — `mission.waypoint.execute` | `stopTask` (pauses the mission) |
| **Simulator** `1.3.3` | **Command-based** — `mission.waypoint.execute` | `mission.pause` / `mission.resume` commands |
| **SAPIENT** `1.3.0` | Task-based — its own protocol owns the task | via task methods |
| **Betaflight**, **RNS** | No waypoint mission support | — |

Both paths are configured with the **same** `WaypointTaskConfig` object (see
[Configuration](#configuration)). Only the delivery differs: the task-based path persists it on a
Task record that the adapter fetches, and the command-based path sends it inline with the command.

## Path A — command-based (MAVLink, Simulator)

No Mission or Task record is created. The waypoints and configuration travel inside a single
command:

```java
import com.zqnt.sdk.client.remotecontrol.domains.CustomCommandRequest;
import com.zqnt.utils.JsonUtils;
import com.zqnt.utils.missionautonomy.domains.WaypointDTO;
import com.zqnt.utils.missionautonomy.domains.config.WaypointTaskConfig;
import com.fasterxml.jackson.core.type.TypeReference;

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

Map<String, Object> params = JsonUtils.getMapper()
        .convertValue(config, new TypeReference<Map<String, Object>>() {});

var request = CustomCommandRequest.builder()
        .sn("SIM-DRONE-001")                      // the Asset SN — see "Which SN" below
        .commandType("mission.waypoint.execute")
        .params(params)
        .build();

var response = client.remoteControl().sendCustomCommand(request).join();
if (!response.isSuccess()) {
    log.warn("Rejected: {}", response.getError().getErrorMessage());
}
```

Only `sn` and `commandType` are required on the request; `tid` is generated when omitted.

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

### Pause and resume (Simulator)

Sent as their own commands, with no params:

```java
client.remoteControl().sendCustomCommand(
    CustomCommandRequest.builder().sn(sn).commandType("mission.pause").build());

client.remoteControl().sendCustomCommand(
    CustomCommandRequest.builder().sn(sn).commandType("mission.resume").build());
```

Pause freezes the aircraft in place; resume continues the same leg. Both are idempotent — pausing
an already-paused mission returns success, not an error, so a retried request is safe. Both are
**rejected when no mission is active**, which includes the window after the final waypoint when the
aircraft has already begun its automatic return home.

MAVLink does not expose these commands. Use `stopTask` there, which pauses the running mission.

## Path B — task-based (DJI)

The DJI adapter fetches the Task from the Connector, builds the KMZ from its config, uploads it and
executes it on the dock. Create the Task first, then start it:

```java
WaypointTaskConfig config = WaypointTaskConfig.builder()
        .waypoints(List.of(/* ... as above ... */))
        .globalSpeed(5.0f)
        .globalHeight(50.0f)
        .build();

TaskDTO task = TaskDTO.builder()
        .name("Perimeter sweep")
        .missionId(missionId)
        .snNumber("DOCK-SN")
        .taskType(TaskTypeProto.TASK_TYPE_WAYPOINT)   // required — see below
        .config(config)
        .build();

var created = client.missionAutonomy().createTask(task).join();
client.missionAutonomy().startTask(created.getTaskId());
```

`pauseTask(taskId)`, `resumeTask(taskId)` and `stopTask(taskId)` control it from there.

> **`taskType` must be `TASK_TYPE_WAYPOINT` and `config` must be a `WaypointTaskConfig`.** The DJI
> adapter checks this explicitly and rejects anything else with `Task type is not WAYPOINT`. A Task
> created as `TASK_TYPE_CUSTOM_COMMAND`, or with a null `config`, cannot be started.

`startTask` runs `prepareTask` first, so you do not need to call it yourself.

## Configuration

Both paths use `WaypointTaskConfig`. Per-waypoint entries (`WaypointDTO`) go in the required
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

A started flight reports progress as **task notifications**, on both paths. Subscribe with
`streamNotifications` and read `getTaskEvent()`:

```java
var notifyRequest = new StreamNotificationRequest();
notifyRequest.setSn("DOCK-SN");

client.liveData().streamNotifications(notifyRequest, n -> {
    var task = n.getTaskEvent();
    if (task == null) return;

    log.info("{} {} {}%", task.getTaskId(), task.getStatus(),
             task.getProgress() == null ? 0 : Math.round(task.getProgress() * 100));
});
```

`TaskEvent` carries `taskId`, `taskType`, `status`, `progress` (0.0–1.0) and `message`. `status` is
a `TaskStatus` — `TASK_RUNNING`, `TASK_PAUSED`, `TASK_COMPLETED`, `TASK_ERROR` and so on.

| Adapter | Progress reporting |
| --- | --- |
| **DJI** `1.3.0` | Full. Derived from the dock's own `flighttask_progress`, with a real percentage. Also reports take-off and fly-to progress |
| **MAVLink** `1.3.0` | `TASK_RUNNING` with `current / total` waypoints, then `TASK_COMPLETED`; `TASK_ERROR` on failure |
| **Simulator** `1.3.3` | **None.** The simulator publishes telemetry only and emits no notifications at all |

> **`TASK_COMPLETED` means every waypoint was reached, not that the aircraft has landed.** With the
> default `WF_ACTION_GO_HOME` finish action it is normally still flying home when that event
> arrives. Watch telemetry if you need the actual landing.

**Developing against the simulator:** the notification stream stays silent for a mission that is
running perfectly. Infer progress from telemetry instead — position advancing through the legs, and
`mode` returning to `ASSET_MODE_IDLE` once the flight ends. Expect real events only on hardware.

## Checking what an asset supports

```java
client.remoteControl().getCapabilities(sn)
```

reports the commands an asset advertises. Note that an adapter's advertised input schema may be
deliberately permissive and list fewer properties than it actually accepts — the tables above are
authoritative for `WaypointTaskConfig`.

## See also

- [Remote Control](REMOTE_CONTROL.md) — the rest of the direct command surface
- [Connector](CONNECTOR.md) — Mission and Task records
- [Assets & Sub-Assets](../concepts/assets-and-sub-assets.md) — which SN to address
