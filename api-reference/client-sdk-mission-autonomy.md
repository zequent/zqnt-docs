# Zequent Client SDK — Mission Autonomy API Reference

> For the conceptual introduction to this model, see
> [Applications & Skills](../concepts/applications-and-skills.md). For 1.3.x (end of life), see the
> [1.3 Mission Autonomy reference](client-sdk-mission-autonomy-1.3.md).

Exhaustive method reference for `client.missionAutonomy()`. Mission/Task CRUD is
**gone** — replaced by capability package (`Application`) administration and capability execution
(`SkillExecution`). Scheduler CRUD *methods* keep the same shapes, but `SchedulerDTO` itself
doesn't — see [Schedulers](#schedulers--same-methods-different-schedulerdto-shape) below.

## What changed from 1.3

- `createMission`/`updateMission`/`getMission`/`deleteMission`/`uploadMissionNfzZones` and every
  Task method (`createTask`, `startTask`, `pauseTask`, ...) **still exist on the interface**, kept
  as `@Deprecated` stubs — but every one of them now returns
  `CompletableFuture.failedFuture(new UnsupportedOperationException(...))` immediately. They don't
  reach the backend at all; there is no RPC left to call. Route optimization, NFZ expansion, and
  the whole task execution lifecycle described in the 1.3.x reference are gone along with them.
- `deleteAllSchedulersByTaskId` was **removed outright** — it isn't on the interface at all (not
  even as a deprecated stub). The remaining Scheduler methods keep the same signatures,
  but what a schedule points at changes — see [Schedulers](#schedulers--same-methods-different-schedulerdto-shape)
  below.
- Two new families replace what Mission/Task did: **Application** (capability package admin) and
  **SkillExecution** (capability execution — create, start, pause, resume, cancel, signal, query).

## Applications (capability package administration)

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsertApplication(ApplicationProtoDTO, expectedRevision)` | `ApplicationProtoDTO` | Create or update an Application. `expectedRevision` is optimistic-concurrency — pass the revision you last read, or `null` on first create |
| `getApplication(applicationId, version)` | `ApplicationProtoDTO` | Get one version of an Application, or its current version if `version` is `null` |
| `listApplications(ApplicationQuery)` | `ResultPage<ApplicationProtoDTO>` | List/filter Applications. `ApplicationQuery.firstPage()` for an unfiltered first page |
| `deleteApplication(applicationId, version, expectedRevision)` | `void` | Delete one version of an Application |

`upsertApplication` can succeed with non-fatal `warnings` attached (e.g. a graph node unreachable
from the start node) — the save still goes through; check the response's warnings so an editor can
surface what's incomplete without blocking a work-in-progress graph from being saved.

There is **no** `getApplicationEnvironments`/`promoteApplicationVersion` on the Java interface — the
Python Client SDK has both (see the
[Python reference](client-sdk-mission-autonomy-python.md#applications-capability-package-administration));
from Java, promote a version in the Admin Console. This is a real difference between the SDKs, not
a doc omission.

## SkillExecution — create/execute

| Method | Returns | Purpose |
| --- | --- | --- |
| `createSkillExecution(SkillExecutionCommand)` | `SkillExecutionProtoDTO` | Create an execution. Does **not** guarantee an atomic plan+start the way `executeSkill` does |
| `executeSkill(SkillExecutionCommand)` | `SkillExecutionProtoDTO` | Convenience operation — create, plan, and (if `options.autoStart`) start an execution atomically in one RPC |

Both take the same `SkillExecutionCommand` record
(`assetSn`, `spec`, `options`, `idempotencyKey`, `organizationId`, `locationId`, `theatreId`).

`organizationId` is checked against the caller's credential, not just recorded: omit it and it's
filled in from the credential's organization; assert a *different* organization and the RPC fails
with `PERMISSION_DENIED` instead of being honoured. `system_admin` callers and platform services
without an organization may assert any organization. Same rule filters `listSkillExecutions`
below: a caller bound to an organization asking for everything is answered with only that
organization's executions, regardless of what `organizationId` they pass.

Build `spec` with one of its two static factories rather than by hand — both set
`options.autoStart = true` by default:

- `SkillExecutionCommand.simple(assetSn, commandId, CapabilityTarget, Struct parameters, idempotencyKey)`
  — a single ad-hoc command, run through the execution engine.
- `SkillExecutionCommand.packaged(assetSn, applicationId, skillId, applicationVersion, Struct parameters, idempotencyKey)`
  — a named Skill from a deployed Application. `applicationVersion` `null` runs the version
  promoted to Production, or the newest version if none is promoted.

`idempotencyKey` defaults to a random UUID when `null`/blank on either factory — repeated requests
with the same asset and key are guaranteed to return the original execution, not create a second
one.

## SkillExecution — query and lifecycle

| Method | Returns | Purpose |
| --- | --- | --- |
| `getSkillExecution(executionId)` | `SkillExecutionProtoDTO` | Get one execution by ID |
| `listSkillExecutions(SkillExecutionQuery)` | `ResultPage<SkillExecutionProtoDTO>` | List/filter executions (by asset, org, status, applicationId, skillId, theatreId). `SkillExecutionQuery.firstPage()` for unfiltered |
| `startSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Start an execution created via `createSkillExecution` without auto-start |
| `pauseSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Pause a running execution |
| `resumeSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Resume a paused execution |
| `cancelSkillExecution(SkillExecutionLifecycleCommand)` | `SkillExecutionProtoDTO` | Cancel an execution |
| `signalSkillExecution(SkillExecutionSignalCommand)` | `SkillExecutionProtoDTO` | Advance an `EVENT_WAIT` node (`eventType` + optional `data`) or resolve a `HUMAN_APPROVAL` node (`approved`) |

`SkillExecutionLifecycleCommand.forExecution(executionId)` builds a bare lifecycle command with no
`reason`/`idempotencyKey`; construct the record directly to set either.
`SkillExecutionSignalCommand` requires at least one of `eventType` or `approved` — the constructor
throws `IllegalArgumentException` if both are absent.

## Execution configuration

| Method | Returns | Purpose |
| --- | --- | --- |
| `resolveExecutionConfig(ExecutionConfigQuery)` | `ResolvedExecutionConfigProtoDTO` | Resolve effective config values (and where each came from) for a given asset + context + set of keys |

`ExecutionConfigQuery` requires `assetSn`, a non-null `context` (`ExecutionConfigContextProto`),
and at least one non-blank key.

## Error handling — this is the one convention that doesn't carry over

1.3.x's `MissionResponse`/`TaskResponse`/`SchedulerResponse` never throw for a business-level
failure — they carry `isSuccess()`/`getError()` instead (see
[Functional Responses](../client-sdk/FUNCTIONAL_RESPONSES.md)). **Application and SkillExecution
RPCs do not follow that convention.** They return the raw payload DTO
(`ApplicationProtoDTO`, `SkillExecutionProtoDTO`, ...) directly on success — there's no wrapper to
carry a failure inline, so a failed RPC completes its `CompletableFuture` exceptionally with
`MissionAutonomyClientException` (carrying `getErrorCode()`/`getTransactionId()`) instead.
`Scheduler` methods are unaffected — they still return `SchedulerResponse` with the usual
`isSuccess()`/`getError()` convention, since the CRUD methods themselves didn't change.

## Schedulers — same methods, different `SchedulerDTO` shape

| Method | Returns | Purpose |
| --- | --- | --- |
| `createScheduler(SchedulerDTO)` | `SchedulerResponse` | Create a scheduler |
| `updateScheduler(schedulerId, SchedulerDTO)` | `SchedulerResponse` | Update a scheduler |
| `getScheduler(schedulerId)` | `SchedulerResponse` | Get a scheduler by ID |
| `deleteScheduler(schedulerId)` | `SchedulerResponse` | Delete a scheduler |
| `createSchedulers(List<SchedulerDTO>)` | `SchedulerResponse` | Create several schedulers in one call |
| `deleteSchedulers(List<String>)` | `SchedulerResponse` | Delete several schedulers in one call |

`deleteAllSchedulersByTaskId` is gone — see [What changed from 1.3](#what-changed-from-13)
above. The method signatures are otherwise identical to 1.3.x, but `SchedulerDTO` itself isn't:
`missionId`/`taskId` are gone (`reserved` in the 2.0 protocol, not merely deprecated), replaced with
a direct capability-execution target —

| Field | Type | Notes |
| --- | --- | --- |
| `assetSn` | `String` | Which asset the schedule fires against |
| `commandId` | `String` | Set together with `assetSn` alone for a single ad-hoc command — mutually exclusive with `capabilityPackageId`/`capabilityId` |
| `capabilityPackageId` / `capabilityId` | `String` | Set together (with `assetSn`) to schedule a named Skill from a deployed Application instead — the SDK's own field names for what the wire protocol calls `application_id`/`skill_id` |
| `executionParametersJson` | `String` | Execution parameters as a JSON string, not a `Struct` |
| `autoStart` | `Boolean` | Whether the resulting execution starts immediately |
| `organizationId` | `String` | The organization the schedule belongs to; `null` = system-wide. This SDK does not set it |

`SchedulerDTO.validate()` enforces the mutual exclusivity: exactly one of (`commandId`) or
(`capabilityPackageId` + `capabilityId`) must be set alongside `assetSn`, or it throws
`IllegalArgumentException` before any RPC is made.

The platform itself accepts an Application schedule without an asset and picks one at firing time
(see [Scheduled triggers](../concepts/applications-and-skills.md#scheduled-triggers)), but
`validate()` still requires `assetSn`, so from this SDK every schedule names its asset. Scheduled
runs have priority 50, and every firing's outcome is recorded on the schedule (visible in the Admin
Console); `SchedulerDTO` does not carry those firing fields.

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction and
  runnable examples
- [Python reference](client-sdk-mission-autonomy-python.md)
- [Go — running and controlling executions](../concepts/applications-and-skills.md#go)
- [1.3 Mission Autonomy reference](client-sdk-mission-autonomy-1.3.md) — the end-of-life 1.3
  Mission/Task/Scheduler interface
