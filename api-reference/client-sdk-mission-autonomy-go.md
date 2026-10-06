# Zequent Client SDK (Go) — Mission Autonomy API Reference

> For the conceptual introduction, see [Applications & Skills](../concepts/applications-and-skills.md).
> For 1.3.x (end of life), see the [1.3 Go Mission Autonomy reference](client-sdk-mission-autonomy-go-1.3.md).

Method reference for `missionautonomy.New(conn)` (package
`github.com/Zequent/zqnt-client-sdk-go/v2/missionautonomy`; dial mission-autonomy-service, default
port 8004). For Java, see [client-sdk-mission-autonomy.md](client-sdk-mission-autonomy.md); for
Python, [client-sdk-mission-autonomy-python.md](client-sdk-mission-autonomy-python.md).

Every method takes a `context.Context` first and returns `(result, error)`. There is no separate
`HasErrors` flag to check: a platform-side error comes back as a non-nil `error` carrying the
platform's message, and so does a transport failure. A refusal of the credential keeps its gRPC code
(`status.Code(err)` is `codes.Unauthenticated` or `codes.PermissionDenied`).

Types come from two generated packages:

```go
import (
    execdto "github.com/Zequent/zqnt-client-sdk-go/v2/gen/execution/contracts/proto" // requests
    execution "github.com/Zequent/zqnt-client-sdk-go/v2/gen/execution/dto/proto"    // DTOs
)
```

A client credential (`ZQNT_CLIENT_TOKEN`) acts for its own organization: it sees and runs that
organization's Applications and runs only. Schedules and event triggers are managed in the Admin
Console.

## Applications

| Method | Returns | Purpose |
| --- | --- | --- |
| `UpsertApplication(ctx, app *execution.ApplicationProtoDTO, expectedRevision string)` | `*execution.ApplicationProtoDTO` | Create or update an Application. `expectedRevision` guards against a concurrent update: pass the revision you last read, or `""` to skip the check |
| `GetApplication(ctx, applicationID, version string)` | `*execution.ApplicationProtoDTO` | One version of an Application; `""` for the latest |
| `ListApplications(ctx, scope *execution.ApplicationScopeProtoDTO, enabledOnly bool, pageSize int32, pageToken string)` | `([]*execution.ApplicationProtoDTO, nextPageToken string, error)` | List Applications. `scope` `nil` for all; `pageSize` `0` and `pageToken` `""` for the defaults |
| `DeleteApplication(ctx, applicationID, version, expectedRevision string)` | `error` | Delete an Application version. `version` and `expectedRevision` are optional (`""`) |

Promoting a version to Production is done in the Admin Console (or from the
[Python client](client-sdk-mission-autonomy-python.md)).

## Running a Skill

Each run is a **Skill execution**. It runs either a named Skill of an Application, or a single
command.

| Method | Returns | Purpose |
| --- | --- | --- |
| `ExecuteApplication(ctx, assetSn, applicationID, skillID, applicationVersion string, parameters *structpb.Struct, idempotencyKey string)` | `*execution.SkillExecutionProtoDTO` | Create and start a run of one Skill from an Application |
| `CreateApplicationExecution(ctx, assetSn, applicationID, skillID, applicationVersion string, parameters *structpb.Struct, idempotencyKey string)` | `*execution.SkillExecutionProtoDTO` | Create the same run without starting it; start it with `StartSkillExecution` |
| `ExecuteSimple(ctx, assetSn, commandID string, parameters *structpb.Struct, idempotencyKey string)` | `*execution.SkillExecutionProtoDTO` | Create and start a run of a single command (e.g. `navigation.go_to`) on one asset |
| `CreateSimpleExecution(ctx, assetSn, commandID string, parameters *structpb.Struct, idempotencyKey string)` | `*execution.SkillExecutionProtoDTO` | Create the same run without starting it |

- **`assetSn`**: required for a single command. For an Application run, `""` lets the platform
  choose: the asset the Application is pinned to, else an online asset of its theatre (site), else
  one picked by your organization's operational policies.
- **`applicationVersion`**: `""` runs the version promoted to Production, or the newest version if
  none is promoted.
- **`parameters`**: the Skill's input (`$.input.<field>` in its mappings) or the command's
  parameters, as a `structpb.Struct`.
- **`idempotencyKey`**: repeating a call with the same asset and key returns the original run
  instead of starting a second one.

## Querying and controlling a run

| Method | Returns | Purpose |
| --- | --- | --- |
| `GetSkillExecution(ctx, executionID)` | `*execution.SkillExecutionProtoDTO` | One run: status, progress and the state of each node |
| `ListSkillExecutions(ctx, query *execdto.ListSkillExecutionsRequest)` | `([]*execution.SkillExecutionProtoDTO, nextPageToken string, error)` | List runs. Filter with the request's optional `AssetSn`, `Status`, `ApplicationId`, `SkillId`, `TheatreId`, `PageSize`, `PageToken`; `nil` lists everything your credential may see |
| `StartSkillExecution(ctx, executionID)` | `*execution.SkillExecutionProtoDTO` | Start a run created without starting |
| `PauseSkillExecution(ctx, executionID)` | `*execution.SkillExecutionProtoDTO` | Pause a running run |
| `ResumeSkillExecution(ctx, executionID)` | `*execution.SkillExecutionProtoDTO` | Resume a paused run |
| `CancelSkillExecution(ctx, executionID)` | `*execution.SkillExecutionProtoDTO` | Cancel a run |
| `SignalSkillExecution(ctx, executionID, nodeID, eventType string, data *structpb.Struct, approved *bool)` | `*execution.SkillExecutionProtoDTO` | Resolve a waiting node: an `EVENT_WAIT` node with `eventType` (+ optional `data`), or a `HUMAN_APPROVAL` node with `approved`. `nodeID` is optional (`""`) |

A human-approval rejection makes the run take its FAILURE path; no answer before the node's timeout
takes its TIMEOUT path — see [Applications & Skills](../concepts/applications-and-skills.md).

## Execution configuration

| Method | Returns | Purpose |
| --- | --- | --- |
| `ResolveExecutionConfig(ctx, execContext *execution.ExecutionConfigContextProto, keys []string)` | `*execution.ResolvedExecutionConfigProtoDTO` | The effective configuration values for a context (asset, theatre, ...) and where each came from. `keys` empty resolves every known key |

## Example

```go
conn, err := grpc.NewClient("localhost:8004",
    append(auth.DialOptions(""), grpc.WithTransportCredentials(insecure.NewCredentials()))...)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()
ma := missionautonomy.New(conn)

input, _ := structpb.NewStruct(map[string]any{"altitude": 40})
run, err := ma.ExecuteApplication(ctx, "", "patrol-app", "patrol", "", input, "patrol-2026-10-05")
if err != nil {
    log.Fatal(err)
}
fmt.Println(run.GetId(), run.GetAssetSn(), run.GetStatus())
```
