# Edge SDK (Python) — Connector API Reference (2.0.x Beta)

> **Beta — not yet released.** Everything on this page describes real, working code — confirmed
> directly against source — but it lives on the unmerged `refactoring/refactoring-edge-sdk-v2`
> branch, not on `main`/the current 1.3.x release. There is no released version number for this
> model yet; treat everything below as subject to change before it ships. If you're building
> against the current 1.3.x platform, see the
> [1.3.x Connector reference](edge-sdk-python-connector-reference.md) instead. For the conceptual
> introduction to this model, see [Applications & Skills](../concepts/applications-and-skills-2.0.md).

Exhaustive method reference for `ConnectorClient` on this branch.

## What actually changed vs. 1.3.x

- **`get_mission`/`get_task`/`get_task_by_flight_id` are gone — not deprecated, just removed.**
  There's no backend RPC left for any of them; the methods don't exist on this branch's
  `ConnectorClient` at all (this SDK doesn't keep the stub-that-raises pattern the client SDK's
  Python `MissionAutonomyClient` uses for its own retired Mission/Task methods — it just deletes
  them outright, matching the Java Edge SDK's clean-removal approach). The 1.3.x reference already
  found [no confirmed real-adapter usage](edge-sdk-python-connector-reference.md#missions-and-tasks)
  of any of the three, so this is unlikely to affect a real adapter migrating forward.
- **New: Skill Registry self-reporting** — four methods, working with the raw generated
  `SkillContractProtoDTO` rather than a plain-Python model (the contract shape is already large and
  typed; wrapping it a second time buys little for what's normally a write-once-per-command call).
- Assets and lifecycle methods are unchanged.

## Assets, lifecycle (unchanged from 1.3.x)

See the [1.3.x reference](edge-sdk-python-connector-reference.md#assets) for these.

## Skill Registry — new in 2.0.x

| Method | Returns | Purpose |
| --- | --- | --- |
| `observe_skill_contract(contract)` | `SkillContractProtoDTO \| None` | Upsert `contract` — new for a never-seen `(command_id, schema_version)` pair, or refreshes content/last-seen for one already known |
| `list_skill_contracts(status=None, command_id=None)` | `list[SkillContractProtoDTO]` | List the registry, optionally filtered by `status` (a `SkillContractStatus` enum value). When `command_id` is set, returns that one command's full version history instead — `status` is then ignored |
| `set_skill_contract_status(contract_id, status)` | `SkillContractProtoDTO \| None` | Move a contract through its lifecycle: `ACTIVE` / `DRAFT` / `DEPRECATED` / `RETIRED` |
| `set_skill_contract_permissions(contract_id, required_permissions)` | `SkillContractProtoDTO \| None` | Full replacement, not a merge. **Declarative only — nothing currently enforces this**, confirmed directly in this method's own docstring: forward-prep for a user-level identity/role system that doesn't exist on the platform yet |

## Error handling

All four Skill Registry methods follow the same convention as the rest of this branch's
`ConnectorClient`: a business-level failure (`resp.has_errors`) is logged and the method returns
`None` (or `[]` for `list_skill_contracts`) rather than raising — unlike the client SDK's Beta
`Application`/`SkillExecution` methods, which raise on failure. Transport-level failures still raise
`grpc.aio.AioRpcError`, same as everywhere else in this SDK.

## See also

- [Applications & Skills](../concepts/applications-and-skills-2.0.md) — narrative introduction
- [Java 2.0.x reference](edge-sdk-connector-reference-2.0.md)
- [1.3.x Connector reference](edge-sdk-python-connector-reference.md) — the current, shipped
  interface
