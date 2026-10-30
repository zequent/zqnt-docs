# Bridges, triggers and steps (3.0 preview)

A **bridge** in the Integration Hub connects a source (an OPC UA server, an HTTP API, a WebSocket,
Kafka, the platform's live telemetry) to a sink, with a mapping in between that turns the source's
data into named values. In 3.0 a bridge works with the platform in two directions:

- **The outside world starts a run.** A fire alarm, a machine fault or a gate sensor: the bridge
  notices the change and an **event trigger** starts an Application.
- **A run asks the outside world.** "Is it raining?" before a takeoff: a step in the Application
  calls the bridge, gets the current values back and decides what to do next.

This page describes a preview; details can change until 3.0 is released. Deploying the Hub itself
is described in the [Integration Hub deployment guide](integration-hub.md).

## Credentials

Connectors that need a login (an MQTT broker, an OPC UA server with a user, an HTTP API key) use a
**secret**. Create it in the console under **Manage → Access & Integrations → Secrets → Integration
Hub** and select it in the connector form's **Secret** field. The value is kept in OpenBao and never
shown again. See [Secrets and sign-in](../admin/secrets-and-sign-in.md#what-the-console-shows).

## When a bridge reads its source

How often a bridge reads its source depends on the kind of source and on the bridge's **poll
interval** (`pollIntervalMs`):

| Source | Poll interval set | No poll interval |
| --- | --- | --- |
| Answers each request at once: HTTP / OpenAPI, OPC UA (read), iosys | reads at that interval | **on demand only**: no background reading; it runs when a step calls it |
| Takes inputs (for example the Open-Meteo template's latitude and longitude) | on demand only | on demand only |
| Pushes messages: WebSocket, Kafka, Zequent Telemetry, OPC UA with **Push on change** | not used | reads each message as it arrives |

A bridge that runs on demand shows the status **on demand**. It cannot start Applications by itself:
it never reads anything until something calls it. Give it a poll interval, or use a source that
pushes, when it should start runs.

## Starting an Application from an event (triggers)

1. **Bridge.** Source connector → mapping → sink. The sink is a connector of type **Zequent
   Application (start a Skill)** (`zqnt-application`) with an **empty** configuration: the event
   triggers decide what is started, not the connector.
2. **Mapping.** Give the values clear names, for example `temperature` from an OPC UA node's value.
   Triggers compare these names, not the source's raw field names.
3. **Event trigger.** Under **Manage → Event Triggers**, create a trigger of type **Integration
   bridge**:
   - **Bridge**: the bridge from step 1. A trigger always listens to a bridge, never to a connector;
     the platform refuses a connector here.
   - **Field**, **operator** and **value**: a mapped name and the condition, for example
     `temperature` greater than `80`. Without a field, every message starts the Application.
   - The Application and Skill to start, and a **cooldown** so a value that stays above the limit
     does not start a run with every message.

The bridge needs to read its source by itself for this: a poll interval, or a source that pushes
(see the table above).

## Asking the outside world from a run (bridges as steps)

Every enabled bridge is also a command an Application can run: `integration.bridge.<bridge id>`,
shown as `Integration Hub: <bridge name>`. Only bridges of the Application's own organization
can be used.

- **Reading.** Without parameters, or when the source takes inputs, the step reads the bridge's
  source once, right then (the parameters fill in the inputs), applies the mapping and returns the
  mapped values as the step's output. A condition after it can use
  `$.nodes.<step id>.output.<name>`. If the bridge has a sink, what was read is also written to it.
- **Writing.** With parameters, and a source that takes no inputs, the step writes the parameters
  out through the bridge's sink, for example to raise an alarm in a plant's OPC UA server.

A bridge used only to ask a question needs no poll interval and no sink.

### Example: weather check before takeoff

1. Connector from the template **Open-Meteo (Weather at a position)**: an HTTP source that takes
   `latitude` and `longitude` as inputs, no secret needed.
2. Bridge `weather` from that connector, with a mapping to `precipitation`, `wind_speed` and
   `wind_gusts`, no sink, enabled. It runs on demand.
3. In the Application, before the takeoff:
   - a **Command** step `integration.bridge.<id of weather>` with the parameters `latitude` and
     `longitude` (for example the site's coordinates),
   - a **Condition**: `precipitation` above 0, `wind_speed` above 10 or `wind_gusts` above 12 → end
     the run ("No takeoff: weather"),
   - wind between 7 and 10 → a **Human approval** ("Wind elevated, take off anyway?"),
   - otherwise continue to the takeoff.

The values come from the moment the run reaches the step, also when a trigger or a schedule started
the run.

## OPC UA: react to changes (Push on change)

By default an OPC UA connector reads its nodes when it is asked (at the bridge's poll interval, or
when a step calls it). Switch on **Push on change (subscription)** in the OPC UA source connector to
let the OPC UA server report every change itself:

| Setting | Meaning |
| --- | --- |
| **Push on change** (`subscribe`) | the server reports changes; the bridge passes each one on as it arrives |
| **Sampling interval** (`samplingIntervalMs`, default 500) | how often the server checks the nodes for changes, in milliseconds |
| **Deadband** (`deadband`, optional) | the smallest change of a numeric value that is reported; filters measurement noise |

Each change sends all of the connector's node values, with the changed one updated, in the same
shape as a read. Existing mappings keep working. A bridge on such a connector ignores its poll
interval, and its triggers react within the sampling interval instead of waiting for the next poll.
Called as a step, it returns the current values as before.

## Good to know

- A trigger that never fires: check that it listens to the right bridge, that the field is a mapped
  name, that the bridge reads by itself (status not **on demand**) and that the value really crosses
  the limit.
- The AI assistant builds bridges, triggers and bridge steps the same way when you ask it to.
