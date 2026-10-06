# Upgrading from 1.3

Zequent **2.0.x** is the current long-term-support (LTS) line. **1.3.x has reached end of life**: it
keeps running, but receives no fixes or support. This page lists what changes when you move an
integration or an edge adapter from 1.3 to 2.0. Every 1.3 guide that loses something in 2.0 links
back here instead of repeating it. The 1.3 documentation itself is kept unchanged in the pages ending
in `-1.3`.

## What's changing

The Mission/Task model is replaced by **Application → Skill → SkillExecution → SkillContract** —
see [Applications & Skills](applications-and-skills.md) for the full concept and runnable
examples. What a schedule *points at* changes too — see [Schedules](#schedules) below.

The platform no longer runs work through an adapter's task methods, and uses `stopTask` only to
cancel a running command — see
[Task-based execution is gone](#task-based-execution-is-gone).

## Installing the 2.0 SDKs

| SDK | 2.0 |
| --- | --- |
| Java (Client and Edge) | `client-java-sdk` / `edge-java-sdk` version `2.0.0` |
| Go (Client and Edge) | The modules are now `/v2`: `go get github.com/Zequent/zqnt-client-sdk-go/v2@v2.0.0` and `go get github.com/Zequent/zqnt-edge-sdk-go/v2@v2.0.0`, and every import path gains `/v2` (for example `github.com/Zequent/zqnt-client-sdk-go/v2/missionautonomy`). Without `/v2`, Go keeps installing the 1.3 line, without an error. |
| Python (Client and Edge) | Not on PyPI yet. Install from the Git tags, together with `zqnt-utils`, the Zequent package both SDKs depend on. With uv, under `[tool.uv.sources]`: `zqnt-client-sdk = { git = "https://github.com/zequent/zqnt-client-sdk-python", tag = "v2.0.0" }` (or `edge-python-sdk = { git = "https://github.com/zequent/zqnt-edge-sdk-python", tag = "v2.0.0" }`) and `zqnt-utils = { git = "https://github.com/zequent/zqnt-utils-python", tag = "v2.0.0" }`. |

## Credentials (new in 2.0)

Every call to the platform carries a credential. A 2.0 platform refuses calls without one.

- **Customer applications** (Client SDKs) send a **client credential**, `ZQNT_CLIENT_TOKEN`, created
  in the Admin Console under **Manage → Access & Integrations → Credentials**. See
  [Client SDK Configuration](../client-sdk/CONFIGURATION.md).
- **Edge adapters** (Edge SDKs) send an **edge credential**, `ZQNT_EDGE_TOKEN`, and verify the
  platform's calls into them with the platform's public key, `ZQNT_PLATFORM_PUBLIC_KEY`. See
  [Edge SDK Configuration](../edge-sdk/edge-sdk-configuration.md).

## Per-SDK impact

| SDK | Mission/Task fate | What's new | Reference |
| --- | --- | --- | --- |
| Client SDK (Java) | Kept as `@Deprecated` stubs — every call fails immediately with `UnsupportedOperationException`. The Connector's Mission/Task methods are removed. | Application admin, SkillExecution (create/execute/query/lifecycle/signal), `resolveExecutionConfig`; on the Connector, the Skill Registry methods (`observeSkillContract`, `listSkillContracts`, `setSkillContractStatus`, `setSkillContractPermissions`) | [Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy.md) |
| Client SDK (Python) | Stubs raise `LegacyOperationRemovedError` synchronously | Same as Java, **plus** `get_application_environments`/`promote_application_version` and four convenience wrappers (`execute_simple`, `execute_application`, ...) | [Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy-python.md) |
| Client SDK (Go) | Removed outright | Applications (`UpsertApplication`, `ExecuteApplication`, `ExecuteSimple`, ...), the SkillExecution lifecycle, `ListSchedulers`; on the Connector, the Skill Registry methods; on Remote Control, `GoToWithOptions` (`PlayTTSAudio` is removed) | [Applications & Skills — Go](applications-and-skills.md#go) |
| Edge SDK (Java) `ConnectorService` | Removed outright — not deprecated, not on the interface at all | Skill Registry self-reporting (`observeSkillContract`, `listSkillContracts`, `setSkillContractStatus`, `setSkillContractPermissions`), `ensureAsset`, pairing-code claims (`redeemAssetClaim`, `describeAssetClaim`), `registerMediaFile` | [Connector reference](../api-reference/edge-sdk-connector-reference.md) |
| Edge SDK (Java) `MissionAutonomyService` | Shrinks to `getScheduler` alone — everything else removed outright | Nothing (unchanged, just smaller) | [Mission Autonomy reference](../api-reference/edge-sdk-mission-autonomy-reference.md) |
| Edge SDK (Java) `EdgeAdapterService` / Live Data | Task methods stay on the interface; only the cancel path is still called (see below). `TaskEventData` is replaced by `CommandExecutionEventData` | `cancelExecution` | — |
| Edge SDK (Python) `ConnectorClient` | `get_mission`/`get_task`/`get_task_by_flight_id` removed outright; `register_asset` is replaced by `ensure_asset` | The same four Skill Registry methods, snake_case, and `redeem_asset_claim` | [Connector reference](../api-reference/edge-sdk-python-connector-reference.md) |
| Edge SDK (Python) adapter / Live Data | `publish_task_event` and `TaskEvent` are replaced by `publish_command_execution_event` and `CommandExecutionEvent` | A command registration API on the adapter (`register_command`, `registered_commands`, ...) | — |
| Edge SDK (Python) `MissionAutonomyClient` | No change — this client was already scheduler-lookup-only before 2.0 | Nothing new on this client, but see the Scheduler shape change below | — |
| Edge SDK (Go) | Nothing removed | Credentials (`WithEdgeToken`, `WithPlatformPublicKey`, and a guard that verifies the platform's calls), `PublishCommandExecutionEvent`, notification streams. Its Connector still has no Skill Registry methods. | — |

Not every SDK has every method: only the Python Client SDK can read an Application's environments
and promote a version from code (`get_application_environments`/`promote_application_version`),
and the Go Edge SDK has no Skill Registry methods.

## Task-based execution is gone

In 1.3, waypoint work could reach a device in two ways: **task-based** (the platform calls your
adapter's `prepareTask`/`startTask`, and the adapter looks the task up) or **command-based** (the
platform sends a custom command such as `mission.waypoint.execute` with everything inline). In 2.0
the platform only uses the **command-based** path: it never calls `prepareTask`, `startTask`,
`pauseTask` or `resumeTask`, and it no longer processes task events. The methods are still on the
Java, Python and Go adapter interfaces, so existing adapters compile, but an adapter that relied on
them must take its work from custom commands (capabilities) instead, and report progress with
command execution events.

One task RPC is still used: the platform cancels a running command with **`StopTask`**, passing the
adapter's own execution id (the one it returned when it accepted the command) in place of a task id.
In Java this arrives at `cancelExecution(sn, externalExecutionId)`, whose default calls `stopTask`;
in Python and Go it arrives at `stop_task` / `StopTask`.

## Schedules

A schedule no longer triggers a Mission/Task. It runs a Skill of an Application (or a single
command) directly: `SchedulerDTO`'s `missionId`/`taskId` are retired, replaced by `assetSn`,
`commandId` or the Application/Skill pair, `executionParametersJson` and `autoStart`.

Schedules and event triggers are administration in 2.0: they are managed in the Admin Console, and a
client credential is refused when it tries to create, change or delete one. An edge adapter can read
a schedule — see [Edge SDK Connector reference — Schedules](../api-reference/edge-sdk-connector-reference.md#schedules)
and [Scheduled triggers](applications-and-skills.md#scheduled-triggers).

## Skill Registry, in brief

The Skill Registry is a persisted, de-duplicated catalog of every command an edge adapter has ever
reported — one entry per `(command_id, schema_version)` — distinct from the live capability
snapshot `getCapabilities`/`get_capabilities` already returns. Each entry carries a `status`
(`ACTIVE`/`DRAFT`/`DEPRECATED`/`RETIRED`) and a server-computed `compatibility` verdict
(`NEW`/`COMPATIBLE`/`BREAKING`) comparing it against the previous version of the same command, so a
schema change that would break an existing authored Skill graph is flagged automatically.
`required_permissions` exists on every entry and carries over to new schema versions, but nothing
enforces it yet. See
[Edge SDK Connector — Skill Registry](../api-reference/edge-sdk-connector-reference.md#skill-registry)
for the full method-by-method reference.

## Where to go next

- [Applications & Skills](applications-and-skills.md) — the concept guide, with runnable
  Java, Python and Go examples
- [Client SDK Mission Autonomy reference](../api-reference/client-sdk-mission-autonomy.md) (Java) /
  [Python](../api-reference/client-sdk-mission-autonomy-python.md)
- [Edge SDK Connector reference](../api-reference/edge-sdk-connector-reference.md) (Java) /
  [Python](../api-reference/edge-sdk-python-connector-reference.md)
- [Edge SDK Mission Autonomy reference](../api-reference/edge-sdk-mission-autonomy-reference.md) (Java)
