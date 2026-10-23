# Applications & Skills

Applications and Skills are how you build multi-step, autonomous behavior on Zequent without writing a state machine by hand for every workflow. If direct commands (takeoff, go-to, open cover) are the platform's verbs, Skills are how you compose them into sentences — and Applications are how you package, version, and deploy those sentences. This model replaces the 1.3.x Mission/Task model for defining automated behavior. Scheduler CRUD itself (`createScheduler`/`getScheduler`/etc.) keeps the same method shapes, but confirmed directly against the protocol and both SDKs, what a schedule *points at* changes: `SchedulerDTO`'s `missionId`/`taskId` fields are retired (`reserved` in the 2.0.x proto, not just deprecated) and replaced with a direct capability-execution target — an asset plus either a single command or an Application+Skill pair, along with execution parameters and an auto-start flag. A schedule now fires a Skill execution directly instead of triggering a Mission/Task. The Java and Python client SDKs disagree on field names for this: Java's `SchedulerDTO` uses `assetSn`/`commandId`/`capabilityPackageId`/`capabilityId`/`executionParametersJson` (a JSON string) while Python's uses `asset_sn`/`command_id`/`application_id`/`skill_id`/`execution_parameters` (a dict) — same concept, different names, not yet reconciled between the two SDKs.

## Concepts

| Term | What it is |
| --- | --- |
| **Skill** | A graph of steps describing one automated behavior — e.g. "patrol perimeter and report," "inspect roof and land." Authored visually as a node graph in the Admin Console. |
| **Application** | A versioned, named bundle of one or more Skills. Applications are what you deploy and promote across environments (development → staging → production). |
| **Skill Execution** | One run of a Skill, usually against a specific asset (or one the platform chooses; see [Which asset an execution runs on](#which-asset-an-execution-runs-on)). Created, started, tracked, and controlled through the Client SDK or the Admin Console. |
| **Skill Contract** | The set of commands (and their schemas) that a given asset/adapter actually supports, self-reported by each edge adapter and used to validate Skills at authoring time. |

## Building a Skill

Skills are authored in the Admin Console's graph editor. A Skill graph is built from nodes:

| Node type | Purpose |
| --- | --- |
| **Command** | Executes a single device command (e.g. `flight.takeoff`, `navigation.go_to`, `dock.open_cover`) against the target asset. |
| **Skill** | Calls another Skill as a sub-step, so common sequences can be reused across Applications. |
| **Condition** | Branches the graph based on a boolean expression (e.g. battery level, telemetry value). |
| **Parallel Gateway / Join Gateway** | Fans work out into concurrent branches and joins them back together. |
| **Wait** | Pauses for a fixed duration before continuing. |
| **Event Wait** | Pauses until an external event (e.g. an asset state change, a detection) signals it forward. |
| **Human Approval** | Pauses until a human operator approves continuation from the Admin Console. |
| **End** | Terminates the graph. |

The graph starts at the first node with no incoming edge unless you choose a different **start node** in the editor. A node that no path reaches is never scheduled: it is marked skipped, and the execution can still succeed without it. A parameter value that arrives as text (from an Integration Hub bridge, an MQTT payload, a CSV column) is converted to the type the command expects, for example `"41.01"` to a number, instead of being refused. For a published Application, the editor marks nodes that have gone stale: a command that was retired, a provider that stopped publishing, a contract that drifted incompatibly, or a node reading an output field its upstream command no longer produces.

Each node can also set a **failure strategy** (`STOP`, `RETRY`, `CONTINUE` or `COMPENSATE`) and a **timeout** in seconds, from the node's settings in the editor.

You don't need to write any of this by hand — the graph editor validates each Command node's parameters against the target asset's live Skill Contract, so you find out about an unsupported command while authoring, not at execution time.

## Which asset an execution runs on

A run whose steps need no asset has none. That is a run of nothing but [Integration Hub](../integrations/integration-hub.md#applications-of-nothing-but-bridges-run-without-an-asset) bridges and platform-run steps such as `route.plan_path`, approvals, waits and conditions. No asset is selected or held for it, it is never refused because an asset is busy, and an asset named on the request is ignored. Its idempotency key is unique within its organization. Its safety checks that are about an asset (telemetry, capabilities, launch limits) do not apply; the organization's risk and schema-drift rules do.

For every other run, an explicit asset serial number on the request always wins. When the request names none, the platform resolves one from the Application, most specific first, and each step runs only if the one before it found nothing:

1. **Asset scope** — an `ASSET` scope entry pins the Application to one known asset.
2. **Theatre scope** — a `THEATRE` scope entry picks among that theatre's assigned assets (online, highest battery).
3. **Policy** — with nothing pinned, the platform selects from every registered asset using your policies, in priority order. The Skill's required asset capabilities filter the candidates, and a location in the execution parameters lets a `NEAREST` strategy rank them. If a policy cannot choose (for example `NEAREST` with no location), the next policy is tried.

Every execution has a priority: a manual run from the Admin Console or the SDK is 0, a scheduled run is 50, and an event trigger is 75 unless it sets its own. Selection prefers a free asset. A busy asset is only offered to a run that is more important than the work it is doing, and then the one doing the least important work. When every available asset is busy with work at least as important, the run is refused as busy. When an execution that commands an asset reaches dispatch, it takes that asset over and cancels the execution that held it. Two executions that only wait on events from the same asset never conflict.

An Application that pins no asset, declares no theatre and gets no asset from any policy fails with an error that names each asset and why it was not used (for example "busy with priority-75 work (this run: 50)", "offline (no telemetry in the last minute)" or "Battery 15% below minimum 20%"). When no policy matches at all, the error says how many policies are active and what kept them out. The platform seeds an **Any Available Asset Fallback** policy (global, most battery, available and at least 20% battery, priority 10) so a run is not left without a policy. A Skill that needs a specific drone can still declare its scope in the editor.

## Scheduled triggers

A **scheduled trigger** starts an Application's Skill at set times. Schedules are managed in the Admin Console (the Triggers tab of the Application or Skill editor, and the Operate → Scheduler calendar).

- **One-time or recurring.** A one-time trigger fires once at a date and time. A recurring trigger fires hourly, daily, on chosen weekdays, or on a day of the month, between a start and an end date, both inclusive. Times are in the trigger's own time zone. Trigger names are unique per Application.
- **Asset.** Pick an asset, or leave it to the platform: the run then follows [Which asset an execution runs on](#which-asset-an-execution-runs-on) at firing time. A single raw command still needs an asset.
- **Organization.** A trigger belongs to an organization and only that organization sees it. An organization admin's trigger is always their own organization's; a system admin may choose one.
- **Inputs.** A trigger must supply every input its Skill needs. Saving one that does not is refused, naming the missing inputs.
- **Every firing is recorded** on the trigger: `STARTED` with the execution it created, `SKIPPED` (no asset could take the run, or the Application is paused) or `FAILED`, each with the reason. A firing missed while the platform was not running is recorded as skipped, and it is never run late.

## Event triggers

An **Event Trigger** starts an Application's Skill when something happens, without your own code calling the SDK. Triggers are managed in the Admin Console (the Triggers tab of the Application or Skill editor) and under `/api/admin-console/event-triggers`. Each trigger carries a target Application and Skill, optional execution parameters, an auto-start flag, and a cooldown in seconds between firings.

| Event type | Fires when |
| --- | --- |
| `DETECTION` | An asset reports a detection, optionally filtered by object type and minimum confidence. |
| `TELEMETRY_THRESHOLD` | A telemetry field (for example `batteryPercentage`) meets a comparison (`LESS_THAN`, `GREATER_THAN`, `EQUALS`, `NOT_EQUALS`). |
| `ASSET_STATUS` | A status field (for example `isOnline`, `operationalMode`) meets a comparison. |
| `WEBHOOK` | An external system POSTs to the trigger's webhook URL, `POST /api/admin-console/event-triggers/webhook/{token}`. The token is generated by the platform and is the only authentication. A JSON object body becomes the execution's parameters, layered over the trigger's own. |
| `INTEGRATION` | A message arrives on an [Integration Hub](../integrations/integration-hub.md#the-reverse-direction-a-source-triggering-a-skill) bridge (`bridgeId`), and the comparison holds over the bridge's mapped payload. |

Leave the asset blank on a trigger to watch any registered asset. A trigger does not need an asset to fire: with none, the execution follows [Which asset an execution runs on](#which-asset-an-execution-runs-on), so an alarm can start a Skill without naming a drone in advance.

## Running a Skill from your application

Once an Application is deployed, trigger one of its Skills against an asset from your own code using the Client SDK's Mission Autonomy client (Java, Python or Go). The 1.3 Mission/Task methods are gone: in Java they remain only as `@Deprecated` stubs that fail immediately (`createMission`, `createTask`, `startTask`, ...), Python raises `LegacyOperationRemovedError` for them, and the Go SDK no longer has them.

### Java

```java
import com.zqnt.sdk.client.missionautonomy.capabilities.SkillExecutionCommand;

// Run a single ad-hoc command through the execution engine (adds tracking/lifecycle
// on top of a plain RemoteControl call):
var adHoc = SkillExecutionCommand.simple(
        "YOUR_DEVICE_SN",
        "flight.takeoff",
        target,          // CapabilityTarget — which asset/sub-asset/payload this targets
        parameters,       // google.protobuf.Struct — command parameters
        null);            // idempotency key — auto-generated if null

// Run a named Skill from a deployed Application:
var skillRun = SkillExecutionCommand.packaged(
        "YOUR_DEVICE_SN",
        "perimeter-patrol-app",  // applicationId
        "patrol-and-report",     // skillId
        null,                    // applicationVersion — null runs the version promoted to Production (the newest if none is)
        parameters,
        null);

client.missionAutonomy().executeSkill(skillRun)
    .thenAccept(execution -> System.out.println("Execution: " + execution.getId()));
```

### Python

```python
# Run a named Skill from a deployed Application:
execution = await client.mission_autonomy.execute_application(
    asset_sn="YOUR_DEVICE_SN",
    application_id="perimeter-patrol-app",
    skill_id="patrol-and-report",
    parameters={"altitude": 60},
)

# Run a single ad-hoc command through the execution engine:
execution = await client.mission_autonomy.execute_simple(
    asset_sn="YOUR_DEVICE_SN",
    command_id="flight.takeoff",
    parameters={"altitude": 60},
)
```

### Go

```go
import (
    "github.com/Zequent/zqnt-client-sdk-go/v2/missionautonomy"
    "google.golang.org/protobuf/types/known/structpb"
)

ma := missionautonomy.New(conn)
params, _ := structpb.NewStruct(map[string]any{"altitude": 60})

// Run a named Skill from a deployed Application ("" as the version runs the version promoted to
// Production, or the newest if none is; "" as the idempotency key sends none):
execution, err := ma.ExecuteApplication(ctx, "YOUR_DEVICE_SN", "perimeter-patrol-app", "patrol-and-report", "", params, "")

// Run a single ad-hoc command through the execution engine:
execution, err = ma.ExecuteSimple(ctx, "YOUR_DEVICE_SN", "flight.takeoff", params, "")
```

All three SDKs can also create an execution without starting it. Only the Python SDK can read an
Application's environments and promote a version from code (`get_application_environments`,
`promote_application_version`); with Java and Go, promote in the Admin Console.

## Tracking and controlling an execution

Every execution has a lifecycle: created → running → (paused) → completed / failed / cancelled.

| Operation | Java | Python | Go |
| --- | --- | --- | --- |
| Get current status | `client.missionAutonomy().getSkillExecution(id)` | `client.mission_autonomy.get_skill_execution(id)` | `ma.GetSkillExecution(ctx, id)` |
| List executions | `client.missionAutonomy().listSkillExecutions(query)` | `client.mission_autonomy.list_skill_executions(...)` | `ma.ListSkillExecutions(ctx, query)` |
| Pause | `client.missionAutonomy().pauseSkillExecution(...)` | `client.mission_autonomy.pause_skill_execution(id)` | `ma.PauseSkillExecution(ctx, id)` |
| Resume | `client.missionAutonomy().resumeSkillExecution(...)` | `client.mission_autonomy.resume_skill_execution(id)` | `ma.ResumeSkillExecution(ctx, id)` |
| Cancel | `client.missionAutonomy().cancelSkillExecution(...)` | `client.mission_autonomy.cancel_skill_execution(id)` | `ma.CancelSkillExecution(ctx, id)` |
| Signal (advance an Event Wait / Human Approval node) | `client.missionAutonomy().signalSkillExecution(...)` | `client.mission_autonomy.signal_skill_execution(...)` | `ma.SignalSkillExecution(ctx, id, nodeID, eventType, data, approved)` |

Progress updates (node started/completed/failed, pause/resume, completion) are also streamed through the Live Data service, so a long-running Skill's progress can be shown live in your own UI the same way telemetry is.

### Watching executions in the Admin Console

The Admin Console receives every execution event as it happens over a WebSocket (`/ws/admin-console/executions`) instead of polling a list. Executions started by a schedule or an event trigger produce the same events as ones started by hand.

| Event | Notification |
| --- | --- |
| Execution failed, step failed, approval needed | Bell entry and a toast |
| Execution started, completed, cancelled | Bell entry |

Starting an execution from the console no longer navigates anywhere: a toast says it started and opens the execution in a new tab when clicked. Under **Alerting & Rules** you can switch toasts on or off separately from the bell, switch each event type, enable an alert sound (off by default), and mute an asset or an application.

On an execution's page, **Live View** shows the asset on a map and marks the steps as they start: takeoff and home, go-to, look-at, and the numbered points of a waypoint mission joined by a line. Finished steps stay, dimmed; a failed step is red. Takeoff and home are placed where the aircraft was when the takeoff was first seen, so they are missing if the page is opened long after takeoff.

### Approvals

A Human Approval node parks the execution. The status stays `RUNNING`, and the platform publishes a `BLOCKED` event. Who may decide depends on the node's **approval group**. Without a group, any signed-in user can approve or reject. With a group, only users who hold a role of that exact name, or a system admin, can decide; the platform refuses anyone else, and the console shows them **Waiting for approval by `<group>`** instead of the buttons. The group is matched against the platform roles (`org-admin`, `operator`, `approver`, `viewer`), exactly and without the usual role ladder: a group of `approver` lets approvers decide, not org-admins. Any other name means only a system admin can decide. Decide from the toast (it stays until someone decides), the header's "Approval needed" indicator, or the execution page; the toast closes everywhere once one person has decided. If the node has a timeout and nobody answers, it follows its `TIMEOUT` edges, or its failure edges when it has none. A failed node whose failure branch ran to completion no longer marks the whole execution failed.

## Simulating an Application (3.0 preview)

> **Preview.** Simulations are part of Zequent **3.0**, which is not released yet; they need a
> platform that runs 3.0.

A simulation runs an Application before it flies, with mock data instead of devices. It is not the
fleet simulator: no simulated device is started, and no adapter is involved.

- **Same engine.** The run goes through the platform's own execution engine: graph, mappings,
  branches, joins, approvals, FAILURE and TIMEOUT edges, asset selection, policies, sites and safety
  rules all behave as in a real run. Only the last step of dispatch differs: no command leaves the
  platform. Each command is answered from its published contract — an output generated from its
  output schema, completing on the reply or later, as the contract says — or from your scenario.
- **Nothing reaches a device.** A simulated run never holds, commands or changes a real asset, and
  never takes an asset over from another run.
- **Scenarios per node.** By default every command succeeds after 10 seconds of simulated time,
  and every approval is approved. For a node, or for every node sending one command, you can set
  instead: success, failure with an error code (for example `flight.not_airborne`, which takes the node's
  FAILURE edges), timeout (the command never answers, so the node takes its TIMEOUT edges), a delay,
  fixed output values that drive a branch (for example a battery of 18 %), and for a Human Approval
  approve, reject (a FAILURE) or no answer (a TIMEOUT). An Event Wait's scenario says when its event
  arrives and with which payload; an AI node answers from its scenario or from its output schema, and
  the AI assistant is never asked.
- **A virtual fleet.** The platform chooses the run's asset from the simulation's own fleet: assets
  you make up (type, published commands, position, battery, site, online), or copies of your
  organization's real assets as the platform last heard of them — nothing is sent to the original.
  Without a fleet, the simulation uses a copy of every aircraft and dock of the organization. Each
  virtual asset runs under a serial number of its own, `VIRTUAL-<run>-<sn>`. An Application pinned to
  a real asset runs on that asset's copy.
- **Fast-forward or real time.** By default waits, timeouts and command durations are skipped: a
  30-minute flight is simulated in moments, and the trace still shows the flight's own timings. A
  simulation can run in real time instead, and can be cancelled while it runs.
- **Marked as a simulation.** A simulated run is kept with the real ones and marked as a simulation
  in the history and the audit log (mode `SIMULATION`). It is left out of execution statistics and
  does not count against license limits. In the Admin Console API, the executions list filters by
  mode: `GET /api/admin-console/applications/executions?mode=SIMULATION` (or `LIVE`; without `mode`
  both).

**Permissions.** Starting and cancelling a simulation need `executions:simulate`, reading its trace
`executions:read`. A person needs them organization-wide or for the site of the run: the operator
and organization admin roles have both, viewers and approvers can read traces. A client credential
has both for its own organization. Every call acts only on your organization's Applications and
runs; another organization's Application or run is not found.

**Starting one.** From your own code through the Client SDK — see
[Simulating an Application](../client-sdk/SIMULATION-3.0.md) for Java, Python and Go. The Admin
Console API offers the same as `POST /api/admin-console/applications/executions/simulations`
(permission `executions:simulate`; a request without simulation settings uses the defaults above).

### Reading the trace

The trace is the path the run took through its graph, in the order its nodes started. Read it while
the run is going or once it has ended; for a simulation every timing in it is simulated time. For
each node it shows:

- its status and attempt, and the asset that performed it — or the asset it was dispatched to, and
  why;
- what went in (the command's parameters after mapping) and what came out, or the error;
- when it started, relative to the run's start, and how long it took;
- **why the run went on**: each edge taken, with its type (normal, success, failure, timeout,
  compensation) and a reason in words such as `condition true`, `failed: flight.not_airborne` or
  `timed out`;
- for a simulation, **what decided its outcome**: your scenario, or an output generated from the
  command's output schema.

Nodes that never ran — the branches not taken — are listed separately, and the trace also names the
asset the platform chose and the fleet under the serial numbers the run used. The Admin Console API
serves the trace of any run, live or simulated, as `GET
/api/admin-console/applications/executions/{id}/trace` (permission `executions:read`). The scenario,
clock and outcome details of a simulation are kept for seven days.

### Simulator first for the AI assistant

An organization can require that Application versions the AI assistant built prove themselves in a
simulation before they run on a real asset. An organization admin switches this on under **Manage →
AI Assistant → Simulator first** (permission `agent:configure`); it applies to the whole organization
and is off until someone switches it on. While it is on, a version the assistant saved or changed
runs on a real asset only after one run of that version succeeded as a simulation or on an asset of
the platform's fleet simulator. Until then:

- a start on a real asset is refused (`FAILED_PRECONDITION`) with a message naming the version and
  what to do;
- a schedule's firing is recorded as `SKIPPED` with that reason, and an event trigger does not start
  the run;
- simulations, and runs on the fleet simulator's assets, start as usual.

Versions people saved are not affected.

## Managing Applications

Applications themselves — creating, versioning, and promoting them between environments (`DEVELOPMENT` / `STAGING` / `PRODUCTION`) — are managed from the Admin Console, and are also available programmatically for CI/CD-style deployment pipelines:

```java
client.missionAutonomy().upsertApplication(applicationDefinition, expectedRevision);
client.missionAutonomy().listApplications(query);
client.missionAutonomy().getApplication(applicationId, version);
```

In the Admin Console, Applications and Skills can also be deleted, and the Executions list shows which Application each run belongs to and where it stopped. Selecting a node on an execution shows what it received and returned.

Deleting an Application without naming a version deletes its latest version.

Schedules and event triggers point at an Application, not at one version, so deleting some versions leaves them running on the newest remaining one. Deleting the **last** version switches them off. They are kept, not deleted: the console marks them **Application deleted** and offers **Delete**.

Most integrations only need the read/execute side (running Skills, checking their status) shown above — authoring and promoting Applications is normally a one-time or occasional workflow done visually in the Admin Console.

## See also

- [No-fly zones and safe returns](airspace-safety.md) — how every flight is checked against your
  no-fly zones, and the platform's low-battery return
- [Technical configuration and dispatch rules](configuration.md) — run settings per organization or
  site, and the rules that choose an asset for a run that names none
- [Java Client SDK Quickstart](../client-sdk/QUICKSTART.md)
- [Python Client SDK Quickstart](../client-sdk/QUICKSTART_PYTHON.md)
- [Go Client SDK Quickstart](../client-sdk/QUICKSTART_GO.md)
- [Client SDK — Mission Autonomy Reference (Java)](../api-reference/client-sdk-mission-autonomy.md)
- [Client SDK — Mission Autonomy Reference (Python)](../api-reference/client-sdk-mission-autonomy-python.md)
- [Simulating an Application](../client-sdk/SIMULATION-3.0.md) — simulations and traces from the
  Java, Python and Go Client SDKs (3.0 preview)
- [Edge SDK — Connector Reference (Java)](../api-reference/edge-sdk-connector-reference.md) — the
  Skill Registry self-reporting API (`observeSkillContract` and friends) an edge adapter uses to push its
  own command contracts, beyond what it already reports live via `getCapabilities`
- [Edge SDK — Connector Reference (Python)](../api-reference/edge-sdk-python-connector-reference.md)
