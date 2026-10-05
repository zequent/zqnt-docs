# Edge SDK — Mission Autonomy API Reference

> For 1.3.x (end of life), see the
> [1.3 Mission Autonomy reference](edge-sdk-mission-autonomy-reference-1.3.md).

`MissionAutonomyService` has one method:

```java
public interface MissionAutonomyService {
    CompletableFuture<SchedulerDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

`createMission`/`updateMission`/`getMission`/`getTask`/`getTaskByFlightId` are **gone, not
deprecated** — the underlying gRPC methods no longer exist, so there's nothing left to stub. Given
the 1.3 interface already had
[no confirmed real-adapter usage](edge-sdk-mission-autonomy-reference-1.3.md) of any of them, this is
unlikely to affect a real adapter migrating forward.

## Schedulers

| Method | Returns | Purpose |
| --- | --- | --- |
| `getScheduler(GetSchedulerRequest)` | `SchedulerDTO` | Get a scheduler by ID (`request.getSchedulerId()`) |

A call that fails in transport is retried before it fails; an error response is logged and resolves
to `null`. `ConnectorService.getSchedulerById` reads a scheduler too, and scheduler
create/update/delete exist only on `ConnectorService` — this interface has no write methods for
schedulers at all. See the [Connector reference](edge-sdk-connector-reference.md#schedulers--same-methods-different-schedulerdto-shape).

`getScheduler` returns the reshaped `SchedulerDTO` — see the
[Client SDK reference](client-sdk-mission-autonomy.md#schedulers--same-methods-different-schedulerdto-shape)
for the full field-by-field breakdown (the client and edge SDKs share the same DTO).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction
- [Connector reference](edge-sdk-connector-reference.md)
- [1.3 Mission Autonomy reference](edge-sdk-mission-autonomy-reference-1.3.md) — the end-of-life 1.3
  interface
