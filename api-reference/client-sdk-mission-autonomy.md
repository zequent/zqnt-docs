# Zequent Client SDK — Mission Autonomy API Reference

> For the conceptual introduction, see [Applications & Skills](../concepts/applications-and-skills.md).
> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Mission Autonomy reference](client-sdk-mission-autonomy-1.3.md).

Method reference for `client.missionAutonomy()`. For Python, see
[client-sdk-mission-autonomy-python.md](client-sdk-mission-autonomy-python.md); for Go,
[client-sdk-mission-autonomy-go.md](client-sdk-mission-autonomy-go.md).

Types used here are the generated protos (`ApplicationProtoDTO`, `SkillExecutionProtoDTO`, ...)
and small records in `com.zqnt.sdk.client.missionautonomy.capabilities`. Every method returns a
`CompletableFuture` that completes with the DTO on success, and completes exceptionally with
`MissionAutonomyClientException` (`getErrorCode()`, `getTransactionId()`) when the platform answers
with an error. A refusal of the credential completes exceptionally with a gRPC
`StatusRuntimeException` (`UNAUTHENTICATED` or `PERMISSION_DENIED`) whose message says what to do.

A client credential (`ZQNT_CLIENT_TOKEN`) acts for its own organization: it sees and runs that
organization's Applications and runs only.

## Applications

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsertApplication(ApplicationProtoDTO, String expectedRevision)` | `ApplicationProtoDTO` | Create or update an Application. `expectedRevision` guards against a concurrent update: pass the revision you last read, or `null` on first create |
| `getApplication(String applicationId, String version)` | `ApplicationProtoDTO` | One version of an Application; `null` for the latest |
| `listApplications(ApplicationQuery)` | `ResultPage<ApplicationProtoDTO>` | List Applications (`scope`, `enabledOnly`, `pageSize` 1–200, `pageToken`). `ApplicationQuery.firstPage()` for an unfiltered first page |
| `deleteApplication(String applicationId, String version, String expectedRevision)` | `Void` | Delete an Application version |

Promoting a version to Production is done in the Admin Console (or from the
[Python client](client-sdk-mission-autonomy-python.md)).

## Running a Skill

Each run is a **Skill execution**. It runs either a named Skill of an Application, or a single
command.

| Method | Returns | Purpose |
| --- | --- | --- |
| `executeSkill(SkillExecutionCommand)` | `SkillExecutionProtoDTO` | Create, plan and start a run in one call |
| `createSkillExecution(SkillExecutionCommand)` | `SkillExecutionProtoDTO` | Create a run without starting it; start it with `startSkillExecution` |

Build the `SkillExecutionCommand` with one of its factories:

- `SkillExecutionCommand.packaged(assetSn, applicationId, skillId, applicationVersion, Struct parameters, idempotencyKey)`
  — a named Skill of an Application.
- `SkillExecutionCommand.simple(assetSn, commandId, CapabilityTarget target, Struct parameters, idempotencyKey)`
  — a single command (e.g. `navigation.go_to`) on one asset. `target` may be `null`.

The record itself is `SkillExecutionCommand(assetSn, spec, options, idempotencyKey, organizationId,
locationId, theatreId)`; construct it directly to set the last three.

- **`assetSn`**: required for a single command. For an Application run, `null` lets the platform
  choose: the asset the Application is pinned to, else an online asset of its theatre (site), else
  one picked by your organization's operational policies.
- **`applicationVersion`**: `null` runs the version promoted to Production, or the newest version if
  none is promoted.
- **`parameters`**: the Skill's input (`$.input.<field>` in its mappings) or the command's
  parameters.
- **`idempotencyKey`**: `null` generates one. Repeating a call with the same asset and key returns
  the original run instead of starting a second one.
- **`organizationId`**: filled in from your credential. Naming a different organization is refused
  with `PERMISSION_DENIED`.

## Querying and controlling a run

| Method | Returns | Purpose |
| --- | --- | --- |
| `getSkillExecution(String executionId)` | `SkillExecutionProtoDTO` | One run: status, progress and the state of each node |
| `listSkillExecutions(SkillExecutionQuery)` | `ResultPage<SkillExecutionProtoDTO>` | List runs, filtered by `assetSn`, `status`, `applicationId`, `skillId`, `theatreId` (`pageSize` 1–200, `pageToken`). `SkillExecutionQuery.firstPage()` for unfiltered. You only see your own organization's runs |
| `startSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Start a run created with `createSkillExecution` |
| `pauseSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Pause a running run |
| `resumeSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Resume a paused run |
| `cancelSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Cancel a run |
| `signalSkillExecution(SkillExecutionSignalCommand)` | `SkillExecutionProtoDTO` | Resolve a waiting node: an `EVENT_WAIT` node with `eventType` (+ optional `data`), or a `HUMAN_APPROVAL` node with `approved` |

`SkillExecutionLifecycleCommand.forExecution(executionId)` builds a lifecycle command without
`reason`/`idempotencyKey`; construct the record directly to set them.
`SkillExecutionSignalCommand(executionId, nodeId, eventType, data, approved, idempotencyKey)` needs
at least one of `eventType` or `approved`, or the constructor throws `IllegalArgumentException`.

A human-approval rejection makes the run take its FAILURE path; no answer before the node's timeout
takes its TIMEOUT path — see [Applications & Skills](../concepts/applications-and-skills.md).

## Execution configuration

| Method | Returns | Purpose |
| --- | --- | --- |
| `resolveExecutionConfig(ExecutionConfigQuery)` | `ResolvedExecutionConfigProtoDTO` | The effective configuration values for an asset and context, and where each came from |

`ExecutionConfigQuery(assetSn, context, keys)` needs `assetSn`, a non-null
`ExecutionConfigContextProto` and at least one non-blank key.

## Schedules

The interface also carries scheduler methods (`createScheduler`, `updateScheduler`, `getScheduler`,
`deleteScheduler`, `createSchedulers`, `deleteSchedulers`). A client credential is refused
(`PERMISSION_DENIED`): schedules and event triggers are managed in the Admin Console — see
[Scheduled triggers](../concepts/applications-and-skills.md#scheduled-triggers).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction and
  runnable examples
- [Remote Control](client-sdk-remote-control.md) — single commands with typed methods
