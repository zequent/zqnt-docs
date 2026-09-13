# Migrating to 2.0.x (Beta)

> **Beta — not yet released.** Everything on this page describes real, working code — confirmed
> directly against source — but it lives on active, unmerged development branches
> (`refactoring/*-v2` across the protocol, client/edge SDKs, and platform services), not on
> `main`/the current 1.3.x release. There is no released version number for this model yet; treat
> everything below as subject to change before it ships. This page is the single place this
> context lives — every 1.3.x guide that loses something in 2.0.x links back here instead of
> repeating it.

## What's actually changing

The Mission/Task model is replaced by **Application → Skill → SkillExecution → SkillContract** —
see [Applications & Skills](applications-and-skills-2.0.md) for the full concept and runnable
examples. Scheduler CRUD itself doesn't go anywhere, but what a schedule *points at* does — see
[Scheduler shape change](#scheduler-shape-change-affects-every-sdk) below, since it's easy to miss
and affects every SDK, including the one edge client whose method surface doesn't change at all.

**Go is untouched.** Every `v2`/`refactoring-*-v2` branch in both the Go Client SDK and Go Edge SDK
is commit-identical to `main` — confirmed directly, not assumed. If you're on Go, none of this page
applies to you yet.

## Per-SDK impact

| SDK | Mission/Task fate | What's new | Reference |
| --- | --- | --- | --- |
| Client SDK (Java) | Kept as `@Deprecated` stubs — every call fails immediately with `UnsupportedOperationException` | Application admin, SkillExecution (create/execute/query/lifecycle/signal), `resolveExecutionConfig` | [2.0.x reference](../api-reference/client-sdk-mission-autonomy-2.0.md) |
| Client SDK (Python) | Stubs raise `LegacyOperationRemovedError` synchronously | Same as Java, **plus** `get_application_environments`/`promote_application_version` (no Java equivalent yet) and four convenience wrappers (`execute_simple`, `execute_application`, ...) | [2.0.x reference](../api-reference/client-sdk-mission-autonomy-python-2.0.md) |
| Client SDK (Go) | No change — branch is identical to `main` | Nothing | — |
| Edge SDK (Java) `ConnectorService` | Removed outright — not deprecated, not on the interface at all | Skill Registry self-reporting (`observeSkillContract`, `listSkillContracts`, `setSkillContractStatus`, `setSkillContractPermissions`) | [2.0.x reference](../api-reference/edge-sdk-connector-reference-2.0.md) |
| Edge SDK (Java) `MissionAutonomyService` | Shrinks to `getScheduler` alone — everything else removed outright | Nothing (unchanged, just smaller) | [2.0.x reference](../api-reference/edge-sdk-mission-autonomy-reference-2.0.md) |
| Edge SDK (Python) `ConnectorClient` | `get_mission`/`get_task`/`get_task_by_flight_id` removed outright | Same four Skill Registry methods, snake_case | [2.0.x reference](../api-reference/edge-sdk-python-connector-reference-2.0.md) |
| Edge SDK (Python) `MissionAutonomyClient` | No change — this client was already scheduler-lookup-only before 2.0.x | Nothing new on this client, but see the Scheduler shape change below | — |
| Edge SDK (Go) | No change — branch is identical to `main` | Nothing | — |

**Java and Python's Beta surfaces are not in sync with each other yet** — the gaps in the table
above (`get_application_environments`/`promote_application_version`) are real, confirmed by reading
both interfaces directly, not a documentation gap.

## Scheduler shape change (affects every SDK)

Scheduler CRUD methods (`createScheduler`/`getScheduler`/etc.) keep the same signatures everywhere.
What changes is `SchedulerDTO` itself: `missionId`/`taskId` are `reserved` on the 2.0.x wire
protocol — not merely deprecated, permanently retired — replaced with a direct capability-execution
target. A schedule now fires a Skill execution directly instead of triggering a Mission/Task.

The Java and Python SDKs use different field names for the same concept, confirmed against both
languages' real DTOs — not yet reconciled between the two:

| Concept | Java field | Python field |
| --- | --- | --- |
| Target asset | `assetSn` | `asset_sn` |
| Single ad-hoc command | `commandId` | `command_id` |
| Application + Skill pair | `capabilityPackageId` + `capabilityId` | `application_id` + `skill_id` |
| Execution parameters | `executionParametersJson` (JSON string) | `execution_parameters` (dict) |
| Auto-start flag | `autoStart` | `auto_start` |

Java's `SchedulerDTO.validate()` enforces that exactly one of (`commandId`) or
(`capabilityPackageId` + `capabilityId`) is set — Python's model has no equivalent client-side
check; an invalid combination is only caught server-side. Full field-by-field breakdown:
[Client SDK 2.0.x reference — Schedulers](../api-reference/client-sdk-mission-autonomy-2.0.md#schedulers--same-methods-different-schedulerdto-shape).

## Skill Registry, in brief

The Skill Registry is a persisted, de-duplicated catalog of every command an edge adapter has ever
reported — one entry per `(command_id, schema_version)` — distinct from the live capability
snapshot `getCapabilities`/`get_capabilities` already returns. Each entry carries a `status`
(`ACTIVE`/`DRAFT`/`DEPRECATED`/`RETIRED`) and a server-computed `compatibility` verdict
(`NEW`/`COMPATIBLE`/`BREAKING`) comparing it against the previous version of the same command, so a
schema change that would break an existing authored Skill graph is flagged automatically.
`required_permissions` exists on every entry but is **declarative only right now** — there's no
user-level identity/role system on the platform yet, so nothing enforces it. See
[Edge SDK Connector — Skill Registry](../api-reference/edge-sdk-connector-reference-2.0.md#skill-registry--new-in-20x)
for the full method-by-method reference.

## Where to go next

- [Applications & Skills](applications-and-skills-2.0.md) — the concept guide, with runnable
  Java/Python examples
- [Client SDK Mission Autonomy — 2.0.x reference](../api-reference/client-sdk-mission-autonomy-2.0.md) (Java) /
  [Python](../api-reference/client-sdk-mission-autonomy-python-2.0.md)
- [Edge SDK Connector — 2.0.x reference](../api-reference/edge-sdk-connector-reference-2.0.md) (Java) /
  [Python](../api-reference/edge-sdk-python-connector-reference-2.0.md)
- [Edge SDK Mission Autonomy — 2.0.x reference](../api-reference/edge-sdk-mission-autonomy-reference-2.0.md) (Java)
