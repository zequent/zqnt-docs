# Zequent Client SDK (Python) — Mission Autonomy API Reference (2.0.x Beta)

> **Beta — not yet released.** Everything on this page describes real, working code — confirmed
> directly against source — but it lives on the unmerged `refactoring/refactoring-client-sdk-v2`
> branch, not on `main`/the current 1.3.x release. There is no released version number for this
> model yet; treat everything below as subject to change before it ships. If you're building
> against the current 1.3.x platform, see the
> [1.3.x Mission Autonomy reference](client-sdk-mission-autonomy-python.md) instead. For the
> conceptual introduction to this model, see
> [Applications & Skills](../concepts/applications-and-skills-2.0.md).

Exhaustive method reference for `client.mission_autonomy` on this branch. Every method is a
coroutine. Mission/Task methods (`create_mission`, `create_task`, `start_task`, ...) are kept as
stubs that immediately `raise LegacyOperationRemovedError` — there's no backend RPC left for any
of them.

## What actually changed vs. 1.3.x

- Every Mission/Task method now raises `LegacyOperationRemovedError` unconditionally, instead of
  making a (now-nonexistent) RPC. Route optimization, NFZ expansion, and the task execution
  lifecycle described in the 1.3.x reference are gone along with them.
- Scheduler CRUD is unchanged — this SDK never had a `delete_all_schedulers_by_task_id` method to
  begin with (unlike Java's 1.3.x interface, which does), so there's nothing to remove here.
- Two new families replace what Mission/Task did: **Application** (capability package admin) and
  **SkillExecution** (capability execution).
- **Python's Beta surface is more extensive than Java's** — `get_application_environments` and
  `promote_application_version` exist here with no Java equivalent on that branch (confirmed by
  reading both interfaces directly — this is a real, current gap between the two SDKs' Beta work,
  not a doc omission). Everything else lines up 1:1 with the
  [Java 2.0.x reference](client-sdk-mission-autonomy-2.0.md).

## Applications (capability package administration)

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsert_application(application, expected_revision=None)` | `ApplicationProtoDTO` | Create or update an Application. `expected_revision` is optimistic-concurrency — the revision you last read, or omitted on first create |
| `get_application(application_id, version=None)` | `ApplicationProtoDTO` | Get one version, or the current version if `version` is omitted |
| `list_applications(scope=None, enabled_only=None, page_size=None, page_token=None)` | `tuple[list[ApplicationProtoDTO], str]` | `(applications, next_page_token)`. Unfiltered when called with no arguments |
| `delete_application(application_id, version=None, expected_revision=None)` | `None` | Delete one version of an Application |
| `get_application_environments(application_id)` | `list[ApplicationEnvironmentPointerProtoDTO]` | **Python-only.** Which version is deployed to each environment (`DEVELOPMENT`/`STAGING`/`PRODUCTION`) |
| `promote_application_version(application_id, version, environment)` | `list[ApplicationEnvironmentPointerProtoDTO]` | **Python-only.** Repoint one environment at an already-saved `version` — promotion never mutates or creates a version, only which one is current for that environment |

## SkillExecution — create/execute

| Method | Returns | Purpose |
| --- | --- | --- |
| `create_skill_execution(asset_sn, spec, options=None, idempotency_key="", ...)` | `SkillExecutionProtoDTO` | Low-level: create an execution from a raw `spec`/`options` proto. Does not guarantee an atomic plan+start |
| `execute_skill(asset_sn, spec, options=None, idempotency_key="", ...)` | `SkillExecutionProtoDTO` | Low-level: create, plan, and (if `options.auto_start`) start atomically in one RPC |
| `create_simple_execution(asset_sn, command_id, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Convenience: create (don't start) a single ad-hoc command execution |
| `execute_simple(asset_sn, command_id, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Convenience: create and atomically start a single ad-hoc command execution |
| `create_application_execution(asset_sn, application_id, skill_id, application_version=None, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Convenience: create (don't start) an execution of one named Skill from a deployed Application |
| `execute_application(asset_sn, application_id, skill_id, application_version=None, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Convenience: create and atomically start an execution of one named Skill from a deployed Application |

The four convenience methods are what the [Applications & Skills](../concepts/applications-and-skills-2.0.md)
guide's example uses — they're thin wrappers over `create_skill_execution`/`execute_skill` that
build the `spec` for you, functionally equivalent to constructing a Java `SkillExecutionCommand`
via its `.simple()`/`.packaged()` static factories. There's no `idempotency_key` auto-generation
here the way Java's factories default to a random UUID — pass one explicitly if you need
retry-safety.

## SkillExecution — query and lifecycle

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_skill_execution(execution_id)` | `SkillExecutionProtoDTO` | Get one execution by ID |
| `list_skill_executions(asset_sn=None, organization_id=None, status=None, application_id=None, skill_id=None, theatre_id=None, page_size=None, page_token=None)` | `tuple[list[SkillExecutionProtoDTO], str]` | `(executions, next_page_token)`. Unfiltered when called with no arguments |
| `start_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Start an execution created without auto-start |
| `pause_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Pause a running execution |
| `resume_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Resume a paused execution |
| `cancel_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Cancel an execution |
| `signal_skill_execution(execution_id, node_id=None, event_type=None, data=None, approved=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Advance an `EVENT_WAIT` node (`event_type` + optional `data`) or resolve a `HUMAN_APPROVAL` node (`approved`) |

## Execution configuration

| Method | Returns | Purpose |
| --- | --- | --- |
| `resolve_execution_config(context, keys=None)` | `dict` | Resolve effective config values for a given asset + context + set of keys |

## Error handling — this is the one convention that doesn't carry over

1.3.x's `MissionResponse`/`TaskResponse`/`SchedulerResponse` never raise for a business-level
failure — `success`/`error` carry it instead (see
[Mission Autonomy — Error handling](../client-sdk/MISSION_AUTONOMY_PYTHON.md#error-handling)).
**Application and SkillExecution methods do not follow that convention.** They return the raw
payload DTO directly on success — there's no wrapper to carry a failure inline — so a failed RPC
raises `MissionAutonomyError` instead (mirroring Java's `MissionAutonomyClientException`).
`Scheduler` methods are unaffected — they still return `SchedulerResponse` with the usual
`success`/`error` convention, since the CRUD methods themselves didn't change on this branch.

## Schedulers — same methods, different `SchedulerDTO` shape

| Method | Returns | Notes |
| --- | --- | --- |
| `create_scheduler(scheduler)` | `SchedulerResponse` | |
| `update_scheduler(scheduler_id, scheduler)` | `SchedulerResponse` | |
| `get_scheduler(scheduler_id)` | `SchedulerResponse` | |
| `delete_scheduler(scheduler_id)` | `SchedulerResponse` | |
| `create_schedulers(schedulers)` | `SchedulerResponse` | Create several in one call |
| `delete_schedulers(scheduler_ids)` | `SchedulerResponse` | Delete several in one call |
| `list_schedulers()` | `SchedulerResponse` | Result is in `.schedulers` — no filtering parameter on this branch |

The method signatures are identical to 1.3.x, but `SchedulerDTO` itself isn't: `mission_id`/
`task_id` are gone (`reserved` on the wire, not merely deprecated), replaced with a direct
capability-execution target —

| Field | Type | Notes |
| --- | --- | --- |
| `asset_sn` | `str \| None` | Which asset the schedule fires against |
| `command_id` | `str \| None` | Set together with `asset_sn` alone for a single ad-hoc command — exactly one of this or `application_id`+`skill_id` is expected |
| `application_id` / `skill_id` | `str \| None` | Set together (with `asset_sn`) to schedule a named Skill from a deployed Application instead |
| `execution_parameters` | `dict \| None` | Unlike Java's `SchedulerDTO` (a JSON string field), this is a plain dict |
| `auto_start` | `bool \| None` | Whether the resulting execution starts immediately |

Unlike the Java client SDK's `SchedulerDTO.validate()`, nothing in this model enforces the
`command_id` vs. `application_id`+`skill_id` exclusivity client-side — an invalid combination is
only caught server-side.

## See also

- [Applications & Skills](../concepts/applications-and-skills-2.0.md) — narrative introduction and
  runnable examples
- [Java 2.0.x reference](client-sdk-mission-autonomy-2.0.md)
- [1.3.x Mission Autonomy reference](client-sdk-mission-autonomy-python.md) — the current, shipped
  Mission/Task/Scheduler interface
