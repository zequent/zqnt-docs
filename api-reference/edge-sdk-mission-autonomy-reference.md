# Edge SDK — Mission Autonomy API Reference

> Coming from 1.3? See the [Migration guide](../concepts/migration-guide.md); for 1.3.x (end of
> life), the [1.3 Mission Autonomy reference](edge-sdk-mission-autonomy-reference-1.3.md).

`MissionAutonomyService` has one method:

```java
public interface MissionAutonomyService {
    CompletableFuture<SchedulerDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

| Method | Returns | Purpose |
| --- | --- | --- |
| `getScheduler(GetSchedulerRequest)` | `SchedulerDTO` | Read a schedule by ID (`request.getSchedulerId()`) |

A call that fails in transport is retried before it fails; an error response is logged and resolves
to `null`. `ConnectorService.getSchedulerById` reads a schedule too. Schedules are created and
changed in the Admin Console. For the `SchedulerDTO` fields, see the
[Connector reference — Schedules](edge-sdk-connector-reference.md#schedules).

Work reaches an adapter as commands, not through this service — see the
[Edge Adapter reference](edge-sdk-adapter-reference.md).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction
- [Connector reference](edge-sdk-connector-reference.md)
