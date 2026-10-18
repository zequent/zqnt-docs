# Simulating an Application (3.0 preview)

> **Preview.** This page describes the Zequent **3.0** simulation API of the Java, Python and Go
> Client SDKs. 3.0 is not released yet: the API below is available as preview builds and needs a
> platform that runs 3.0. Names may still change before the release.

A simulation runs an Application through the platform's own execution engine — graph, mappings,
branches, joins, approvals, FAILURE and TIMEOUT edges, asset selection, safety rules — on a
**virtual fleet**, and no command leaves the platform. Every command is answered from its published
contract and from the scenario you send with the request. Afterwards the **trace** tells you which
way the run went through its graph and why.

Use it to check an Application before it flies: does the failure branch of a takeoff really land the
aircraft, does a low battery reading take the return-home branch, what happens when nobody answers
an approval. For what a simulation is and what it never touches, see
[Simulating an Application](../concepts/applications-and-skills.md#simulating-an-application-30-preview).

The service behind this page is `zqnt.application_simulation.v3.SimulationService` (package
`zqnt.application_simulation.v3` of the public `zqnt-api` contracts), served by the Mission Autonomy
service:

| RPC | What it does |
| --- | --- |
| `StartSimulation` | Starts a simulated run and answers once it is running, with its execution id and status. The run continues in the platform |
| `GetSimulationTrace` | The path the run took so far, or in full once it has ended |
| `CancelSimulation` | Stops a simulated run, for example one in real time |

## Step 1: SDK versions and connection

The simulation API is in these preview builds or later (install details as on the Commands pages:
[Java](COMMANDS-3.0.md#step-1-add-the-preview-dependency) ·
[Python](COMMANDS_PYTHON-3.0.md#step-1-add-the-preview-dependency) ·
[Go](COMMANDS_GO-3.0.md#step-1-add-the-preview-module)):

| SDK | Version | Entry point |
| --- | --- | --- |
| Java | `com.zqnt:zqnt-sdk-java-client` `3.0.0-PR27-795183b-SNAPSHOT` | `zequent.simulations()` (`com.zqnt.sdk.client.simulation.application.Simulations`) |
| Python | `zqnt-client-sdk` from the `feature/v3-agentic` branch of `zequent/zqnt-sdk-python` | `client.simulations` (`client_sdk.SimulationsClient`) |
| Go | `github.com/zequent/zqnt-sdk-go/v3@feature/v3-agentic` | `simulation.New(conn)` (package `github.com/zequent/zqnt-sdk-go/v3/client/simulation`) |

The request, scenario and trace messages are the generated `zqnt-api` types: Java
`com.zqnt.protos.application_simulation.v3`, Python
`zqnt_api.generated.zqnt.application_simulation.v3.application_simulation_pb2`, Go
`github.com/zequent/zqnt-api/go/v3/gen/zqnt/application_simulation/v3`. In Python your code imports
`zqnt_api` directly, so keep its `[tool.uv.sources]` entry on exactly the revision the SDK pins (see
the SDK's `client/pyproject.toml`).

The simulation client uses the Mission Autonomy connection (`MISSION_AUTONOMY_SERVICE_HOST` /
`_PORT`, default port `8004`) and the same `ZQNT_CLIENT_TOKEN` client credential as the rest of the
SDK (see [Client SDK Configuration](CONFIGURATION.md)). In Java and Python nothing else needs setting
up; in Go, dial it with `cfg.Dial(cfg.MissionAutonomy)`.

**Permissions.** Starting and cancelling a simulation need the permission `executions:simulate`,
reading its trace `executions:read`. A client credential has both for its own organization. Every
call acts only on your organization's Applications and runs.

## Step 2: Describe the run

A `StartSimulationRequest` names the Application and, optionally, how each node behaves and which
assets the platform may choose from. Everything except `application_id` has a default:

| Field | Meaning | Unset |
| --- | --- | --- |
| `context.idempotency_key` | A start retried with the same key answers the run it started — see [Idempotency](#idempotency) | The SDK generates a new key for each call |
| `application_id` | The Application to simulate (required) | — |
| `application_version` | The version to run | The version promoted to Production, else the newest |
| `skill_id` | The Skill of the Application to run | Its first Skill |
| `inputs` | The run's input, read by mappings as `$.input.<field>` | No input |
| `site_id` | Only an asset of this site is chosen | Any asset of the fleet |
| `real_time` | Waits, timeouts and command durations pass in real time | Fast-forward: a 30-minute flight is simulated in moments, and its timings in the trace stay those of the flight |
| `command_duration` | How long a command without a scenario `delay` takes, in simulated time | 10 seconds |
| `approvals` | How a human approval without a scenario of its own is answered | Approved |
| `nodes` | One `NodeScenario` per node or command whose outcome you want to set | Every node succeeds |
| `fleet` | The `VirtualAsset`s asset selection, policies and sites choose from | A copy of every aircraft and dock of your organization |

### Scenarios: `NodeScenario`

A node without a scenario succeeds after `command_duration`, with an output generated from its
command's output schema. A scenario changes that for one node or for every node that sends a
command:

| Field | Meaning |
| --- | --- |
| `node_id` | The node in the Application's graph; a node of a composed Skill is `"<skill node id>/<node id>"`. Wins over `command_id` |
| `command_id` | Every node sending this command (for example `flight.takeoff`), when no scenario names the node itself |
| `outcome` | `NODE_OUTCOME_SUCCESS` (also when unset); `NODE_OUTCOME_FAILURE` follows the node's FAILURE edges with `error_code` / `error_message`; `NODE_OUTCOME_TIMEOUT` never answers, so the node runs into its timeout and follows its TIMEOUT edges |
| `error_code`, `error_message` | For a failure: a stable code such as `flight.not_airborne`, and a text for people |
| `delay` | Simulated time until the outcome: a command's duration, when an approval is answered, when the event an Event Wait waits for arrives, when an AI node answers. Unset = `command_duration` |
| `output` | Laid over the generated output, for example `{"battery_percent": 18}` to drive a branch. For an approval its comment, for an Event Wait the event's payload, for an AI node its answer |
| `approval` | Human Approval nodes: `APPROVAL_ANSWER_APPROVE`, `APPROVAL_ANSWER_REJECT` (follows the FAILURE edges) or `APPROVAL_ANSWER_NO_ANSWER` (the node times out and follows its TIMEOUT edges). Overrides the request's `approvals` |

An AI node answers from its scenario, or with values generated from its output schema; the AI
assistant is never asked.

### The virtual fleet: `VirtualAsset`

Each asset of the fleet is either made up or a copy of one of your organization's real assets
(`copy_of_sn`) as the platform last heard of it. Nothing is sent to the real asset, and the fields
you set override the copy's.

| Field | Meaning | Unset |
| --- | --- | --- |
| `sn` | Unique in the fleet | For a copy: the original's serial number |
| `copy_of_sn` | The serial number of a real asset of your organization | A made-up asset |
| `name` | Display name | The serial number (a copy keeps the original's name) |
| `type` | `ASSET_TYPE_AIRCRAFT`, `_DOCK`, `_SENSOR`, `_CAMERA`, `_JAMMER`, `_OTHER` | An aircraft (a copy: the original's type) |
| `commands` | The command ids it publishes, for example `flight.takeoff`; each gets its contract from the Skill Registry when there is one | For a copy: what the original last published |
| `position` | Where it is; altitude in metres above sea level | For a copy: where the original is |
| `battery_percent` | 0–100 | 100, for a copy the original's |
| `site_id` | The site it is stationed at | For a copy: the original's site |
| `online` | Whether it counts as online | Online |

In the run every virtual asset has a serial number of its own, `VIRTUAL-<run>-<sn>`, so it cannot be
mistaken for a real one; the trace shows these serial numbers. An Application pinned to a real asset
runs on that asset's copy, so the fleet must contain one.

## Step 3: Simulate and print the path

The examples simulate a patrol Application on one made-up drone and a copy of the real dock
`DOCK-1`: every `navigation.go_to` takes twelve minutes, the node `inspect` reports a battery of 18 %,
and nobody answers the approval `approve-landing`. `simulate` starts the run and polls its trace
every second until it has ended, for at most two minutes.

### Java

```java
import com.google.protobuf.Struct;
import com.google.protobuf.Value;
import com.zqnt.protos.application_simulation.v3.ApprovalAnswer;
import com.zqnt.protos.application_simulation.v3.NodeScenario;
import com.zqnt.protos.application_simulation.v3.SimulationTrace;
import com.zqnt.protos.application_simulation.v3.StartSimulationRequest;
import com.zqnt.protos.application_simulation.v3.TraceStep;
import com.zqnt.protos.application_simulation.v3.TraceTransition;
import com.zqnt.protos.application_simulation.v3.VirtualAsset;
import com.zqnt.protos.common.v3.GeoPoint;
import com.zqnt.protos.common.v3.RequestContext;
import com.zqnt.sdk.client.simulation.application.Simulations;

import java.time.Duration;

Simulations simulations = zequent.simulations();

StartSimulationRequest request = StartSimulationRequest.newBuilder()
        .setContext(RequestContext.newBuilder().setIdempotencyKey("patrol-check-2026-10-18"))
        .setApplicationId("perimeter-patrol-app")
        .setInputs(Struct.newBuilder()
                .putFields("altitude", Value.newBuilder().setNumberValue(60).build()))
        .addNodes(NodeScenario.newBuilder()
                .setCommandId("navigation.go_to")
                .setDelay(com.google.protobuf.Duration.newBuilder().setSeconds(12 * 60)))
        .addNodes(NodeScenario.newBuilder()
                .setNodeId("inspect")
                .setOutput(Struct.newBuilder()
                        .putFields("battery_percent", Value.newBuilder().setNumberValue(18).build())))
        .addNodes(NodeScenario.newBuilder()
                .setNodeId("approve-landing")
                .setApproval(ApprovalAnswer.APPROVAL_ANSWER_NO_ANSWER))
        .addFleet(VirtualAsset.newBuilder()
                .setSn("DRONE-A")
                .addCommands("flight.takeoff").addCommands("navigation.go_to").addCommands("flight.return_to_home")
                .setPosition(GeoPoint.newBuilder().setLatitude(52.5200).setLongitude(13.4050).setAltitude(34))
                .setBatteryPercent(64))
        .addFleet(VirtualAsset.newBuilder().setCopyOfSn("DOCK-1"))
        .build();

SimulationTrace trace = simulations
        .simulate(request, Duration.ofSeconds(1), Duration.ofMinutes(2))
        .join();

System.out.println(trace.getStatus() + " on " + trace.getAssetSn()
        + ", " + trace.getDuration().getSeconds() + " s simulated");
for (TraceStep step : trace.getStepsList()) {
    System.out.println("+" + step.getOffset().getSeconds() + " s  " + step.getNodeId()
            + "  " + step.getStatus() + "  (" + step.getSimulatedBy() + ")");
    for (TraceTransition next : step.getNextList()) {
        System.out.println("    -> " + next.getTargetNodeId() + ": " + next.getReason());
    }
}
System.out.println("not taken: " + trace.getSkippedNodeIdsList());
```

`startSimulation(request)` and `awaitSimulation(executionId, interval, timeout)` are the two halves
of `simulate`; `simulationTrace(executionId)` reads the trace once. Every method returns a
`CompletableFuture`.

### Python

```python
from google.protobuf import duration_pb2, struct_pb2
from zqnt_api.generated.zqnt.application_simulation.v3 import application_simulation_pb2 as sim
from zqnt_api.generated.zqnt.common.v3 import common_pb2


def struct(values: dict) -> struct_pb2.Struct:
    result = struct_pb2.Struct()
    result.update(values)
    return result


request = sim.StartSimulationRequest(
    context=common_pb2.RequestContext(idempotency_key="patrol-check-2026-10-18"),
    application_id="perimeter-patrol-app",
    inputs=struct({"altitude": 60}),
    nodes=[
        sim.NodeScenario(command_id="navigation.go_to", delay=duration_pb2.Duration(seconds=12 * 60)),
        sim.NodeScenario(node_id="inspect", output=struct({"battery_percent": 18})),
        sim.NodeScenario(node_id="approve-landing", approval=sim.APPROVAL_ANSWER_NO_ANSWER),
    ],
    fleet=[
        sim.VirtualAsset(
            sn="DRONE-A",
            commands=["flight.takeoff", "navigation.go_to", "flight.return_to_home"],
            position=common_pb2.GeoPoint(latitude=52.5200, longitude=13.4050, altitude=34),
            battery_percent=64,
        ),
        sim.VirtualAsset(copy_of_sn="DOCK-1"),
    ],
)

trace = await client.simulations.simulate(request, interval=1.0, timeout=120)

print(sim.SimulationStatus.Name(trace.status), "on", trace.asset_sn, f"{trace.duration.seconds} s simulated")
for step in trace.steps:
    print(f"+{step.offset.seconds} s  {step.node_id}  {sim.NodeStatus.Name(step.status)}  ({step.simulated_by})")
    for next_step in step.next:
        print(f"    -> {next_step.target_node_id}: {next_step.reason}")
print("not taken:", list(trace.skipped_node_ids))
```

`await start_simulation(request)` and `await wait_for_simulation(execution_id, *, interval=1.0,
timeout=None)` are the two halves of `simulate`; `await simulation_trace(execution_id)` reads the
trace once. `interval` and `timeout` take seconds or a `timedelta`.

### Go

```go
import (
    "fmt"
    "log"
    "time"

    simulationv3 "github.com/zequent/zqnt-api/go/v3/gen/zqnt/application_simulation/v3"
    commonv3 "github.com/zequent/zqnt-api/go/v3/gen/zqnt/common/v3"
    "github.com/zequent/zqnt-sdk-go/v3/client/config"
    "github.com/zequent/zqnt-sdk-go/v3/client/simulation"
    "google.golang.org/protobuf/types/known/durationpb"
    "google.golang.org/protobuf/types/known/structpb"
)

cfg, err := config.FromEnv()
if err != nil {
    log.Fatal(err)
}
conn, err := cfg.Dial(cfg.MissionAutonomy)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()
sims := simulation.New(conn)

inputs, _ := structpb.NewStruct(map[string]any{"altitude": 60})
lowBattery, _ := structpb.NewStruct(map[string]any{"battery_percent": 18})
battery, altitude := 64.0, 34.0

trace, err := sims.Simulate(ctx, &simulationv3.StartSimulationRequest{
    Context:       &commonv3.RequestContext{IdempotencyKey: "patrol-check-2026-10-18"},
    ApplicationId: "perimeter-patrol-app",
    Inputs:        inputs,
    Nodes: []*simulationv3.NodeScenario{
        {CommandId: "navigation.go_to", Delay: durationpb.New(12 * time.Minute)},
        {NodeId: "inspect", Output: lowBattery},
        {NodeId: "approve-landing", Approval: simulationv3.ApprovalAnswer_APPROVAL_ANSWER_NO_ANSWER},
    },
    Fleet: []*simulationv3.VirtualAsset{
        {
            Sn:             "DRONE-A",
            Commands:       []string{"flight.takeoff", "navigation.go_to", "flight.return_to_home"},
            Position:       &commonv3.GeoPoint{Latitude: 52.5200, Longitude: 13.4050, Altitude: &altitude},
            BatteryPercent: &battery,
        },
        {CopyOfSn: "DOCK-1"},
    },
}, time.Second, 2*time.Minute)
if err != nil {
    log.Fatal(err)
}

fmt.Println(trace.GetStatus(), "on", trace.GetAssetSn(), trace.GetDuration().AsDuration(), "simulated")
for _, step := range trace.GetSteps() {
    fmt.Printf("+%s  %s  %s  (%s)\n", step.GetOffset().AsDuration(), step.GetNodeId(), step.GetStatus(), step.GetSimulatedBy())
    for _, next := range step.GetNext() {
        fmt.Printf("    -> %s: %s\n", next.GetTargetNodeId(), next.GetReason())
    }
}
fmt.Println("not taken:", trace.GetSkippedNodeIds())
```

`StartSimulation(ctx, req)` and `WaitForSimulation(ctx, executionID, interval, timeout)` are the two
halves of `Simulate`; `SimulationTrace(ctx, executionID)` reads the trace once. `StartSimulation`
does not change the request you pass in.

## Reading the trace

A `SimulationTrace` lists the nodes that ran, in the order they started:

| Field | Meaning |
| --- | --- |
| `status` | `SIMULATION_STATUS_PENDING`, `_RUNNING`, `_PAUSED`, `_CANCELLING` while it runs; `_SUCCEEDED`, `_FAILED`, `_CANCELLED` or `_TIMED_OUT` once it has ended |
| `asset_sn` | The asset the platform chose for the run (a virtual serial number) |
| `fast_forward` | The run skipped its waits |
| `duration` | From start to end (or to now), in simulated time |
| `steps` | One `TraceStep` per node that ran |
| `skipped_node_ids` | Nodes that never ran: branches not taken |
| `error`, `output` | The run's error, and its output |
| `fleet` | The fleet, under the serial numbers the run used |

Each `TraceStep`:

| Field | Meaning |
| --- | --- |
| `node_id`, `name`, `type` | The node; `type` is `NODE_TYPE_COMMAND`, `_SKILL`, `_CONDITION`, `_PARALLEL_GATEWAY`, `_JOIN_GATEWAY`, `_WAIT`, `_EVENT_WAIT`, `_HUMAN_APPROVAL`, `_END` or `_AI` |
| `command_id` | For command nodes |
| `status`, `attempt` | `NODE_STATUS_SUCCEEDED`, `_FAILED`, `_TIMED_OUT`, `_SKIPPED`, `_CANCELLED`, `_WAITING` (for a person or an event), … and which attempt it was |
| `asset_sn`, `dispatch_reason` | Who performed it: the run's asset, or the one it was dispatched to — then `dispatch_reason` says why, in words |
| `input`, `output` | The command's parameters after mapping, and what it returned |
| `error`, `warnings` | Why it failed (`category`, `code`, `message`); what to know about how it ran without it having failed |
| `offset`, `duration` | From the run's start to this node's start, and how long the node took, in simulated time |
| `started_at`, `completed_at` | Timestamps of the node |
| `next` | Why the run went on: one `TraceTransition` per edge taken, with `target_node_id`, `edge_type` (`EDGE_TYPE_NORMAL`, `_SUCCESS`, `_FAILURE`, `_TIMEOUT`, `_COMPENSATION`) and `reason` in words, for example `succeeded`, `condition true`, `failed: flight.not_airborne`, `timed out` |
| `simulated_by` | What decided the node's outcome, for example `scenario: failure` or that its output was generated from the command's output schema |

To check a branch, look at the step before it: its `next` says which edge was taken and why, and
the nodes of the branch that was not taken are in `skipped_node_ids`. A FAILURE edge from a SKILL
node catches a failure anywhere inside that Skill; a rejected approval is a failure, an approval
nobody answered a timeout.

## Waiting, timeouts and cancelling

`simulate` and the wait methods poll the trace every `interval` until the run is `SUCCEEDED`,
`FAILED`, `CANCELLED` or `TIMED_OUT`, and return that trace **whatever the outcome**: a run that
ended `FAILED` is a normal result, not an error — read `trace.error` and the steps. When the
`timeout` elapses first, the wait fails and **the run keeps going** on the platform:

| SDK | No limit | Timed out |
| --- | --- | --- |
| Java | `timeout` `null` | the future fails with a `CommandException` whose `getCategory()` is `ERROR_CATEGORY_TIMEOUT` |
| Python | `timeout=None` (the default) | `CommandError` with `category` `ERROR_CATEGORY_TIMEOUT` |
| Go | `timeout` `0` (only `ctx` limits it) | an error for which `errors.Is(err, simulation.ErrWaitTimedOut)` holds |

`interval` must be positive. Each SDK also tells you whether a status is final:
`Simulations.isFinal(status)` (Java), `client_sdk.simulations.FINAL_SIMULATION_STATUSES` (Python),
`simulation.Final(status)` (Go).

A fast-forwarded run usually ends within moments. A run with `real_time` set takes as long as the
real flight would, so start it and come back to it, or cancel it:

```java
StartSimulationResponse started = simulations.startSimulation(request.toBuilder().setRealTime(true).build()).join();
// ...
SimulationStatus status = simulations.cancelSimulation(started.getExecutionId()).join();
SimulationTrace trace = simulations.awaitSimulation(started.getExecutionId(), Duration.ofSeconds(1), Duration.ofSeconds(30)).join();
```

```python
started = await client.simulations.start_simulation(request)
# ...
status = await client.simulations.cancel_simulation(started.execution_id)
trace = await client.simulations.wait_for_simulation(started.execution_id, timeout=30)
```

```go
started, err := sims.StartSimulation(ctx, req)
// ...
status, err := sims.CancelSimulation(ctx, started.GetExecutionId())
trace, err := sims.WaitForSimulation(ctx, started.GetExecutionId(), time.Second, 30*time.Second)
```

Cancelling answers the run's status: `SIMULATION_STATUS_CANCELLING` while its simulated commands are
being stopped, then `SIMULATION_STATUS_CANCELLED`.

## Errors

The platform's refusals keep their gRPC status:

| Status | When |
| --- | --- |
| `INVALID_ARGUMENT` | An unusable scenario or fleet — for example two assets of the fleet with the same `sn`, a made-up asset without `sn`, a `copy_of_sn` the platform does not know right now or that is not your organization's, an Application pinned to a real asset the fleet has no copy of, or an empty fleet in an organization with no aircraft or dock to copy. The message says which |
| `NOT_FOUND` | An Application that is not your organization's; for the trace and cancel calls, an execution id that is not a simulation of your organization (the id of a live run included) |
| `PERMISSION_DENIED` | The caller lacks `executions:simulate` (start, cancel) or `executions:read` (trace) |

How each SDK carries it:

- **Java** — every future fails with a `CommandException`
  (`com.zqnt.sdk.client.commands.domains`); `getStatus()` is the gRPC status code, `getCategory()`
  the error category, `getMessage()` the platform's message.
- **Python** — every call raises `CommandError` (`from client_sdk import CommandError`); `status`
  is the `grpc.StatusCode`, `category` the error category.
- **Go** — the error wraps the gRPC error: `status.Code(err)` is `codes.InvalidArgument`,
  `codes.NotFound`, `codes.PermissionDenied`, …

An empty `application_id` or `execution_id` is refused before the call leaves your process:
`IllegalArgumentException` in Java, `ValueError` in Python, an error in Go. Java and Python retry
transient failures with their usual retry and circuit-breaker settings; Go has no built-in retry
(as for every Go client).

## Idempotency

`context.idempotency_key` makes a start safe to retry: a start with a key the platform has already
seen answers the run it started instead of starting a second one. When the request has no key, the
SDK generates a new one for that call; the SDK's own retries of that call reuse it. A retry **you**
make by calling `startSimulation` / `start_simulation` / `StartSimulation` again gets a fresh key and
therefore a second run — set your own key when your code may repeat a start (after a timeout, a
restart, a redelivered message).

## See also

- [Applications & Skills](../concepts/applications-and-skills.md#simulating-an-application-30-preview)
  — what a simulation does and never touches, permissions, the AI assistant's "Simulator first"
- [Commands (Java)](COMMANDS-3.0.md) · [Python](COMMANDS_PYTHON-3.0.md) · [Go](COMMANDS_GO-3.0.md)
  — the 3.0 preview SDKs, installation and error types
- Mission Autonomy reference: [Java](../api-reference/client-sdk-mission-autonomy.md) ·
  [Python](../api-reference/client-sdk-mission-autonomy-python.md) ·
  [Go](../api-reference/client-sdk-mission-autonomy-go.md)
