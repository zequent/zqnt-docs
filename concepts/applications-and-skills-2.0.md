# Applications & Skills

> **Beta — not yet released, Java and Python only.** Everything on this page describes real,
> working code — confirmed directly against source — but it lives on active, unmerged development
> branches (`refactoring/*-v2` across the protocol, client SDKs, and platform services), not on
> `main`/the current 1.3.x release. There is no released version number for this model yet; treat
> everything below as subject to change before it ships. **The Go Client SDK has not started this
> migration at all** — its `v2` branch is currently identical to `main`. If you're building against
> the current 1.3.x platform, see [Waypoint Missions](/docs/sdk/client/waypoint-missions) and the
> Mission Autonomy docs instead — this page is a preview, not a guide for today's integrations.

Applications and Skills are how you build multi-step, autonomous behavior on Zequent without writing a state machine by hand for every workflow. If direct commands (takeoff, go-to, open cover) are the platform's verbs, Skills are how you compose them into sentences — and Applications are how you package, version, and deploy those sentences. This model replaces the 1.3.x Mission/Task model for defining automated behavior. Scheduler CRUD itself (`createScheduler`/`getScheduler`/etc.) keeps the same method shapes, but confirmed directly against the protocol and both SDKs, what a schedule *points at* changes: `SchedulerDTO`'s `missionId`/`taskId` fields are retired (`reserved` in the 2.0.x proto, not just deprecated) and replaced with a direct capability-execution target — an asset plus either a single command or an Application+Skill pair, along with execution parameters and an auto-start flag. A schedule now fires a Skill execution directly instead of triggering a Mission/Task. The Java and Python client SDKs disagree on field names for this: Java's `SchedulerDTO` uses `assetSn`/`commandId`/`capabilityPackageId`/`capabilityId`/`executionParametersJson` (a JSON string) while Python's uses `asset_sn`/`command_id`/`application_id`/`skill_id`/`execution_parameters` (a dict) — same concept, different names, not yet reconciled between the two branches.

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

An explicit asset serial number on the request always wins. When the request names none, the platform resolves one from the Application, most specific first, and each step runs only if the one before it found nothing:

1. **Asset scope** — an `ASSET` scope entry pins the Application to one known asset.
2. **Theatre scope** — a `THEATRE` scope entry picks among that theatre's assigned assets (online, highest battery).
3. **Policy** — with nothing pinned, the platform selects from every registered asset using your policies, in priority order. The Skill's required asset capabilities filter the candidates, and a location in the execution parameters lets a `NEAREST` strategy rank them. If a policy cannot choose (for example `NEAREST` with no location), the next policy is tried.

Selection prefers a free asset and falls back to a busy one rather than refusing to answer an alarm. When an execution that commands an asset reaches dispatch, it takes that asset over and cancels the execution that held it. Two executions that only wait on events from the same asset never conflict.

An Application that pins no asset, declares no theatre and matches no policy fails with an error that says so. A Skill that needs a specific drone can still declare its scope in the editor.

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

Once an Application is deployed, trigger one of its Skills against an asset from your own code using the Client SDK's Mission Autonomy client. The old Mission/Task methods (`createMission`, `createTask`, `startTask`, ...) still exist on this interface for now, but only as `@Deprecated` stubs that fail immediately — the platform doesn't silently ignore them, and they don't work in either model at this point.

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
        null,                    // applicationVersion — latest deployed if null
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

Java and Python's Beta surfaces are not perfectly in sync with each other yet — Python currently
exposes a couple of extra methods (`create_application_execution`, `get_application_environments`,
`promote_application_version`) that don't have a Java equivalent on the branch yet.

## Tracking and controlling an execution

Every execution has a lifecycle: created → running → (paused) → completed / failed / cancelled.

| Operation | Java | Python |
| --- | --- | --- |
| Get current status | `client.missionAutonomy().getSkillExecution(id)` | `client.mission_autonomy.get_skill_execution(id)` |
| List executions | `client.missionAutonomy().listSkillExecutions(query)` | `client.mission_autonomy.list_skill_executions(...)` |
| Pause | `client.missionAutonomy().pauseSkillExecution(...)` | `client.mission_autonomy.pause_skill_execution(id)` |
| Resume | `client.missionAutonomy().resumeSkillExecution(...)` | `client.mission_autonomy.resume_skill_execution(id)` |
| Cancel | `client.missionAutonomy().cancelSkillExecution(...)` | `client.mission_autonomy.cancel_skill_execution(id)` |
| Signal (advance an Event Wait / Human Approval node) | `client.missionAutonomy().signalSkillExecution(...)` | `client.mission_autonomy.signal_skill_execution(...)` |

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

A Human Approval node parks the execution. The status stays `RUNNING`, and the platform publishes a `BLOCKED` event. Any signed-in user can approve or reject from the toast (it stays until someone decides), the header's "Approval needed" indicator, or the execution page, and the toast closes everywhere once one person has decided. If the node has a timeout and nobody answers, it follows its `TIMEOUT` edges, or its failure edges when it has none. A failed node whose failure branch ran to completion no longer marks the whole execution failed.

## Managing Applications

Applications themselves — creating, versioning, and promoting them between environments (`DEVELOPMENT` / `STAGING` / `PRODUCTION`) — are managed from the Admin Console, and are also available programmatically for CI/CD-style deployment pipelines:

```java
client.missionAutonomy().upsertApplication(applicationDefinition, expectedRevision);
client.missionAutonomy().listApplications(query);
client.missionAutonomy().getApplication(applicationId, version);
```

In the Admin Console, Applications and Skills can also be deleted, and the Executions list shows which Application each run belongs to and where it stopped. Selecting a node on an execution shows what it received and returned.

Deleting an Application without naming a version deletes its latest version.

Most integrations only need the read/execute side (running Skills, checking their status) shown above — authoring and promoting Applications is normally a one-time or occasional workflow done visually in the Admin Console.

## See also

- [Java Client SDK Quickstart](/docs/sdk/client/quickstart)
- [Python Client SDK Quickstart](/docs/sdk/client/quickstart-python)
- [Client SDK — Mission Autonomy Reference (Java, Beta)](../api-reference/client-sdk-mission-autonomy-2.0.md)
- [Client SDK — Mission Autonomy Reference (Python, Beta)](../api-reference/client-sdk-mission-autonomy-python-2.0.md)
- [Edge SDK — Connector Reference (Java, Beta)](../api-reference/edge-sdk-connector-reference-2.0.md) — the
  Skill Registry self-reporting API (`observeSkillContract` and friends) an edge adapter uses to push its
  own command contracts, beyond what it already reports live via `getCapabilities`
- [Edge SDK — Connector Reference (Python, Beta)](../api-reference/edge-sdk-python-connector-reference-2.0.md)
