# Edge SDK — Connector API Reference (2.0.x Beta)

> **Beta — not yet released.** Everything on this page describes real, working code — confirmed
> directly against source — but it lives on the unmerged `refactoring/refactoring-egde-sdk-v2`
> branch (yes, that's the real branch name — "egde," not "edge"), not on `main`/the current 1.3.x
> release. There is no released version number for this model yet; treat everything below as
> subject to change before it ships. If you're building against the current 1.3.x platform, see
> the [1.3.x Connector reference](edge-sdk-connector-reference.md) instead. For the conceptual
> introduction to this model, see [Applications & Skills](../concepts/applications-and-skills-2.0.md).

Exhaustive method reference for `ConnectorService` on this branch.

## What actually changed vs. 1.3.x

- **Every Mission and Task method is gone — not deprecated, not stubbed, just gone.** Unlike the
  client SDK's `MissionAutonomy` interface (which keeps `@Deprecated` stubs that fail loudly),
  `ConnectorService` on this branch has no `getMissionById`/`createMission`/`updateMission`/
  `deleteMission`/`getTaskById`/`createTask`/`updateTask`/`deleteTask`/`getTaskByFlightId` at all —
  the methods simply don't exist on the interface. A call site referencing any of them fails to
  compile, not fails at runtime.
- **New: Skill Registry self-reporting.** Four methods let an adapter push its own command
  contracts into the platform's persisted Skill Registry directly, instead of only ever being
  polled indirectly through `EdgeAdapterService#getCapabilities`.
- Assets, Asset payloads, and Organization are unchanged. Scheduler *methods* are too, but not
  `SchedulerDTO` itself — see below.

## Assets, asset payloads, organization (unchanged from 1.3.x)

See the [1.3.x reference](edge-sdk-connector-reference.md#assets) for these — same methods, same
behavior on this branch.

## Schedulers — same methods, different `SchedulerDTO` shape

`getSchedulerById`/`createScheduler`/`updateScheduler`/`deleteScheduler` keep the same signatures
as 1.3.x, and use the same shared `com.zqnt.utils.missionautonomy.domains.SchedulerDTO` class the
Client SDK does — which means they're subject to the exact same reshape:
`missionId`/`taskId` are retired (`reserved` on the wire, not merely deprecated), replaced with a
direct capability-execution target. See the
[Client SDK 2.0.x reference — Schedulers](client-sdk-mission-autonomy-2.0.md#schedulers--same-methods-different-schedulerdto-shape)
for the full field-by-field breakdown.

## Skill Registry — new in 2.0.x

| Method | Returns | Purpose |
| --- | --- | --- |
| `observeSkillContract(SkillContractProtoDTO)` | `SkillContractProtoDTO` | Upsert a contract — new for a never-seen `(command_id, schema_version)` pair, or refreshes content/last-seen for one already known |
| `listSkillContracts(status, commandId)` | `List<SkillContractProtoDTO>` | List the registry, optionally filtered by `SkillContractStatus`. When `commandId` is set, returns that one command's full version history instead — `status` is then ignored, matching the RPC's own semantics. Either argument may be `null` |
| `setSkillContractStatus(id, SkillContractStatus)` | `SkillContractProtoDTO` | Move a contract through its lifecycle: `ACTIVE` / `DRAFT` / `DEPRECATED` / `RETIRED` |
| `setSkillContractPermissions(id, List<String>)` | `SkillContractProtoDTO` | Full replacement, not a merge, of the contract's `requiredPermissions` |

`setSkillContractPermissions` is **declarative only right now** — confirmed against the proto's own
comment: there's no user-level identity/role system on the platform yet (only the
installation-level license lease), so nothing currently enforces the permissions a contract
declares. This is forward-prep for when one exists; permission strings are free-form by design
(e.g. `"mission.launch"`, `"role:pilot"`) rather than drawn from a fixed vocabulary.

`SkillContractProtoDTO` also carries a `compatibility` verdict
(`NEW`/`COMPATIBLE`/`BREAKING`/`UNKNOWN`) computed server-side whenever an observation introduces a
new `schema_version` for an already-known `command_id`, comparing its input/output schema against
the previous version — `BREAKING` means a required input was added/changed, or an existing
input/output property was removed or retyped, so an existing authored Skill graph referencing the
previous version may now be invalid. `compatibilityNotes` carries the human-readable reasons (e.g.
`"required field 'zoom' added"`).

In practice, most adapters won't call these directly — the same live `Capability` snapshot an
adapter already returns from `EdgeAdapterService#getCapabilities` is what the platform's Console
aggregates into the registry automatically via `observeSkillContract`; this API exists for an
adapter (or the Integration Hub, which uses it to mirror configured sinks in) that wants to push a
contract proactively rather than only being observed passively. See
[Edge Adapter Reference — Capability Reporting](edge-sdk-adapter-reference.md#capability-reporting)
for the live-snapshot side of this.

## See also

- [Applications & Skills](../concepts/applications-and-skills-2.0.md) — narrative introduction
- [Python 2.0.x reference](edge-sdk-python-connector-reference-2.0.md)
- [1.3.x Connector reference](edge-sdk-connector-reference.md) — the current, shipped interface
