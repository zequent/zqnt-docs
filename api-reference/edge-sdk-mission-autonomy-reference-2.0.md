# Edge SDK — Mission Autonomy API Reference (2.0.x Beta)

> **Beta — not yet released.** Everything on this page describes real, working code — confirmed
> directly against source — but it lives on the unmerged `refactoring/refactoring-egde-sdk-v2`
> branch, not on `main`/the current 1.3.x release. There is no released version number for this
> model yet; treat everything below as subject to change before it ships. If you're building
> against the current 1.3.x platform, see the
> [1.3.x Mission Autonomy reference](edge-sdk-mission-autonomy-reference.md) instead.

`MissionAutonomyService` shrinks to one method on this branch:

```java
public interface MissionAutonomyService {
    CompletableFuture<SchedulerDTO> getScheduler(GetSchedulerRequest getSchedulerRequest);
}
```

`createMission`/`updateMission`/`getMission`/`getTask`/`getTaskByFlightId` are **gone, not
deprecated** — the underlying gRPC methods no longer exist on this branch, so there's nothing left
to stub. Given the 1.3.x interface already had
[no confirmed real-adapter usage](edge-sdk-mission-autonomy-reference.md) of any of them, this is
unlikely to affect a real adapter migrating forward.

`getScheduler` is unchanged in behavior (still a plain passthrough to `connector-service`, see the
[1.3.x reference](edge-sdk-mission-autonomy-reference.md#schedulers)) but returns the reshaped
`SchedulerDTO` — see
[Connector — Skill Registry](edge-sdk-connector-reference-2.0.md) for the rest of what's new on
this branch, and the
[Client SDK 2.0.x reference](client-sdk-mission-autonomy-2.0.md#schedulers--same-methods-different-schedulerdto-shape)
for the full field-by-field `SchedulerDTO` breakdown (the client and edge SDKs share the same
generated DTO).

## See also

- [Applications & Skills](../concepts/applications-and-skills-2.0.md) — narrative introduction
- [Connector — 2.0.x Beta reference](edge-sdk-connector-reference-2.0.md)
- [1.3.x Mission Autonomy reference](edge-sdk-mission-autonomy-reference.md) — the current, shipped
  interface
