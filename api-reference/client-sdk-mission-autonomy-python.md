# Zequent Client SDK (Python) — Mission Autonomy API Reference

> For the conceptual introduction, see [Applications & Skills](../concepts/applications-and-skills.md).
> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Mission Autonomy reference](client-sdk-mission-autonomy-python-1.3.md).

Method reference for `client.mission_autonomy`. For Java, see
[client-sdk-mission-autonomy.md](client-sdk-mission-autonomy.md); for Go,
[client-sdk-mission-autonomy-go.md](client-sdk-mission-autonomy-go.md).

Every method is a coroutine that works with the generated proto types (`ApplicationProtoDTO`,
`SkillExecutionProtoDTO`, ...) and returns the DTO on success. Errors:

- The platform answered with an error: `client_sdk.exceptions.MissionAutonomyError` (`operation`,
  `error_code`, `error_message`).
- The platform refused the credential: `client_sdk.auth.ZequentAuthError`, a `grpc.aio.AioRpcError`
  with code `UNAUTHENTICATED` or `PERMISSION_DENIED` and a message that says what to do.
- A transient failure that outlasted every retry: `client_sdk.ZequentRetryExhaustedError`.

A client credential (`ZQNT_CLIENT_TOKEN`) acts for its own organization: it sees and runs that
organization's Applications and runs only.

## Applications

| Method | Returns | Purpose |
| --- | --- | --- |
| `upsert_application(application, expected_revision=None)` | `ApplicationProtoDTO` | Create or update an Application. `expected_revision` guards against a concurrent update: pass the revision you last read, or omit it on first create |
| `get_application(application_id, version=None)` | `ApplicationProtoDTO` | One version of an Application; omit `version` for the latest |
| `list_applications(scope=None, enabled_only=None, page_size=None, page_token=None)` | `tuple[list, str]` | `(applications, next_page_token)`. `scope` is an `ApplicationScopeProtoDTO`; no arguments lists everything |
| `delete_application(application_id, version=None, expected_revision=None)` | `None` | Delete an Application version |
| `get_application_environments(application_id)` | `list` | Which version is current in each environment (`STAGING`, `PRODUCTION`) |
| `promote_application_version(application_id, version, environment)` | `list` | Make an already-saved `version` the current one for `environment` (an `ApplicationEnvironmentProto` value, e.g. `APPLICATION_ENVIRONMENT_PRODUCTION`). Promotion never changes or creates a version |

## Running a Skill

Each run is a **Skill execution**. It runs either a named Skill of an Application, or a single
command.

| Method | Returns | Purpose |
| --- | --- | --- |
| `execute_application(asset_sn, application_id, skill_id, application_version=None, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Create and start a run of one Skill from an Application |
| `create_application_execution(asset_sn, application_id, skill_id, application_version=None, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Create the same run without starting it; start it with `start_skill_execution` |
| `execute_simple(asset_sn, command_id, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Create and start a run of a single command (e.g. `navigation.go_to`) |
| `create_simple_execution(asset_sn, command_id, parameters=None, idempotency_key="")` | `SkillExecutionProtoDTO` | Create the same run without starting it |
| `execute_skill(asset_sn, spec, options=None, idempotency_key="", organization_id=None, location_id=None, theatre_id=None)` | `SkillExecutionProtoDTO` | Low level: run a `SkillExecutionSpecProto` you built yourself. Starts the run unless `options.auto_start` is `False` |
| `create_skill_execution(asset_sn, spec, options=None, idempotency_key="", organization_id=None, location_id=None, theatre_id=None)` | `SkillExecutionProtoDTO` | Low level: create the run without starting it |

- **`asset_sn`**: the asset to run on; required.
- **`application_version`**: `None` runs the version promoted to Production, or the newest version
  if none is promoted.
- **`parameters`**: a `dict` — the Skill's input (`$.input.<field>` in its mappings) or the
  command's parameters.
- **`idempotency_key`**: repeating a call with the same asset and key returns the original run
  instead of starting a second one. Pass one when you may retry.
- **`organization_id`**: filled in from your credential. Naming a different organization is refused
  with `PERMISSION_DENIED`.

## Querying and controlling a run

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_skill_execution(execution_id)` | `SkillExecutionProtoDTO` | One run: status, progress and the state of each node |
| `list_skill_executions(asset_sn=None, organization_id=None, status=None, application_id=None, skill_id=None, theatre_id=None, page_size=None, page_token=None)` | `tuple[list, str]` | `(executions, next_page_token)`. `status` is a `SkillExecutionStatusProto` value. You only see your own organization's runs |
| `start_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Start a run created without starting |
| `pause_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Pause a running run |
| `resume_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Resume a paused run |
| `cancel_skill_execution(execution_id, reason=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Cancel a run |
| `signal_skill_execution(execution_id, node_id=None, event_type=None, data=None, approved=None, idempotency_key=None)` | `SkillExecutionProtoDTO` | Resolve a waiting node: an `EVENT_WAIT` node with `event_type` (+ optional `data`), or a `HUMAN_APPROVAL` node with `approved` |

A human-approval rejection makes the run take its FAILURE path; no answer before the node's timeout
takes its TIMEOUT path — see [Applications & Skills](../concepts/applications-and-skills.md).

## Execution configuration

| Method | Returns | Purpose |
| --- | --- | --- |
| `resolve_execution_config(context, keys=None)` | `dict` | The effective configuration values for a context (an `ExecutionConfigContextProto`: asset, theatre, Skill, ...). `keys` omitted resolves every known key |

## Schedules

The client also carries scheduler methods (`create_scheduler`, `update_scheduler`, `get_scheduler`,
`delete_scheduler`, `list_schedulers`, `create_schedulers`, `delete_schedulers`). A client
credential is refused (`PERMISSION_DENIED`): schedules and event triggers are managed in the Admin
Console — see [Scheduled triggers](../concepts/applications-and-skills.md#scheduled-triggers).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction and
  runnable examples
- [Remote Control](client-sdk-remote-control-python.md) — single commands with typed methods
