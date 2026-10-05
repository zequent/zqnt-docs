# Edge SDK (Python) — Mission Autonomy

`MissionAutonomyClient` is a small, focused client: it lets an edge adapter look up a **scheduler**
definition directly. Everything else related to running automated behavior on an asset — receiving
commands, reporting their progress — happens through other parts of the SDK, described below.

For Java, see [edge-sdk-mission-autonomy.md](edge-sdk-mission-autonomy.md).

---

## Scheduler lookup

```python
from edge_sdk import MissionAutonomyClient

client = MissionAutonomyClient(host="localhost", port=8004)   # token defaults to ZQNT_EDGE_TOKEN
await client.connect()
try:
    scheduler = await client.get_scheduler(scheduler_id="scheduler-uuid")
finally:
    await client.close()
```

`EdgeAdapterRuntime` connects one for you as `runtime.mission_autonomy`. A scheduler now targets a
Skill or a single command directly, not a Mission or Task — see
[Upgrading from 1.3 — Scheduler shape change](../concepts/migration-guide.md#scheduler-shape-change-affects-every-sdk)
for the fields.

---

## How Skill executions reach your adapter

Automated work is authored as Applications and Skills (see
[Applications & Skills](../concepts/applications-and-skills.md)). The platform runs a Skill by calling
*into* your `EdgeAdapter`, one command at a time — you don't poll or manage executions yourself:

| Concern | Where it lives |
| --- | --- |
| Receiving a command — typed (`take_off`, `go_to`, ...) or custom (`mission.waypoint.execute`, ...) | `EdgeAdapter`, with custom commands declared through `register_command` — see [Edge Adapter — Custom commands](edge-sdk-python-adapter.md#custom-commands) |
| Reporting a command's progress and outcome | Command execution events through `LiveDataService` — see [Edge Adapter — Reporting progress](edge-sdk-python-adapter.md#reporting-progress-for-long-running-commands) |
| Cancelling a running command | `stop_task`, called with the command's `external_execution_id` — see [Edge Adapter — Tasks](edge-sdk-python-adapter.md#tasks) |
| Declaring which commands your adapter supports | `get_capabilities` on `EdgeAdapter`, and the Skill Registry — see [Connector](edge-sdk-python-connector.md#capabilities-and-the-skill-registry) |
| Authoring Applications and Skills, and running them | The Admin Console and the **Client SDK** |

The 2.0 platform no longer calls `prepare_task` or `start_task` — see
[Upgrading from 1.3](../concepts/migration-guide.md#task-based-execution-is-gone).

---

## Best practices

- **Return quickly.** A long-running command answers at once with an `external_execution_id`, and
  reports its progress with command execution events.
- **End every accepted command with exactly one terminal event** — `SUCCEEDED`, `FAILED` or
  `CANCELLED`. A Skill waiting on a command without one waits until it times out.
- **Make cancellation idempotent.** Cancelling a command that has already finished must be a no-op.
