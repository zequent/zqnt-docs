# Technical configuration and dispatch rules

Zequent 2.0 has two places where an organization tunes how the platform behaves, both under
**Configuration** in the Admin Console:

- **Technical Config**: named settings (keys) such as the no-fly-zone safety margin or the low-battery
  floor, set platform-wide, per organization or per site.
- **Dispatch Rules**: which asset responds when a run does not name one. Older console screens and the
  Admin Console API call them **operational policies** (`/api/admin-console/policies`).

This page explains how both work, lists every runtime key with its default, and ends with recipes.

## Who can change what

| | Org admin | System admin |
| --- | --- | --- |
| See the Configuration screen | Yes | Yes |
| Technical Config entries | Only `ORGANIZATION` (their own organization) and `THEATRE` (a site of their own organization) | Every scope |
| Dispatch rules | Only for their own organization or one of its sites | Every scope |

Every entry that could apply to you is visible, including platform-wide ones you cannot edit. Operators
(users without an admin role) cannot open the screen.

## Technical Config

Each entry has a **Key**, a **Value** with a value type (`STRING`, `INTEGER`, `LONG`, `BOOLEAN`,
`DOUBLE` or `JSON`), a **Scope**, an optional scope target and an **Active** switch. An inactive entry is
ignored.

The console offers the scopes `GLOBAL`, `SERVICE`, `ADAPTER`, `ASSET_TYPE`, `ORGANIZATION` and `THEATRE`.
Every runtime key below exists as a `GLOBAL` entry from the start, holding the value the platform uses
when nothing else is set, so you can see it and override it.

### Which value a run uses

The settings a run uses are worked out once, when the run is created, in this order (first match wins):

1. **A per-run override** sent with the run request (`configOverrides` in the run's options). Safety
   settings may only be overridden by an organization admin or a system admin; see
   [Safety-relevant keys](#safety-relevant-keys).
2. **`THEATRE`**: an entry for the site the run belongs to.
3. **`ORGANIZATION`**: an entry for the run's organization.
4. **The Skill's own default**: a default the Skill (or its Application) sets for the key.
5. **`GLOBAL`**.

A `GLOBAL` value never overrides a default the Skill author chose; an organization or site entry does.
`SERVICE`, `ADAPTER` and `ASSET_TYPE` entries are not used for these keys. The **Scope resolution
chain** on the Technical Config screen lists the entries for a key by scope; it does not show a Skill's
own default.

**The site a run belongs to** is the site named by whatever started it (for example a site-bound
trigger); otherwise the site its Application is deployed to; otherwise the site whose area contains the
run's target position (`latitude`/`longitude` in its inputs). A run with none of these, such as most
commands sent from Remote Control, has no site, and only organization and global entries apply.

Some keys are not part of a run's settings and are read live instead:

- the low-battery return keys `route.safety_return.*`: per aircraft, on every check, from the site the
  asset is stationed at, then the asset's organization, then `GLOBAL`;
- the margin Remote Control uses when it checks a fly-to before sending it: organization, then `GLOBAL`;
- the timing keys marked "GLOBAL only" in the tables below.

### When a change takes effect

A saved change reaches every service within moments, without a redeploy or restart. (If that
notification is missed, the services reload all entries every 10 minutes.)

- Keys that are part of a run's settings apply to **runs created after the change**. A run that is
  already running keeps the settings it was created with.
- Keys read live, including `route.safety_return.*`, apply on their next use, also to aircraft already
  in the air.

## Runtime keys

These keys are created as `GLOBAL` entries with the values shown, which are the values the platform
uses when the key is not set at all. "Scopes" says where setting the key has an effect.

### No-fly zones and safe return

See [No-fly zones and safe returns](airspace-safety-2.0.md) for what these do.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `route.nfz.buffer_m` | 20 | Safety margin in metres every flight keeps around a `HARD_BLOCK` no-fly zone; detours are planned this far outside it. Capped at 1000. | Per run; theatre, organization, global |
| `route.safety_return.enabled` | `true` | Send an airborne aircraft home around the no-fly zones on low battery, before its firmware flies its own straight return. | Live; theatre, organization, global |
| `route.safety_return.floor_percent` | 30 | Battery % at or below which the platform always sends the aircraft home. Keep firmware low-battery thresholds below this. | Live; theatre, organization, global |
| `route.safety_return.reserve_percent` | 10 | Battery % added on top of what the path home around the zones needs. | Live; theatre, organization, global |
| `route.safety_return.cruise_speed_mps` | 10 | Assumed speed home in m/s; the measured ground speed is used when slower. | Live; theatre, organization, global |
| `route.safety_return.drain_percent_per_minute` | 2.5 | Assumed battery drain in %/min until the aircraft's own drain has been measured in flight. | Live; theatre, organization, global |
| `route.safety_return.landing_allowance_percent` | 3 | Battery % allowed for the landing at home. | Live; theatre, organization, global |
| `route.safety_return.descent_speed_mps` | 3 | Assumed descent speed in m/s for the height to lose before landing. | Live; theatre, organization, global |
| `route.safety_return.firmware_margin_percent` | 5 | Act this many % above the return-home battery level the aircraft reports (DJI). | Live; theatre, organization, global |

For the safe-return keys, "theatre" means the site the asset is stationed at, not the site of a run.

### Flight preparation

When switched on, these put extra steps in front of **every** run created in that scope, including
single commands sent from Remote Control. The steps come from the built-in Application
`zqnt.system.flight-preparation` ("Zequent system flight preparation"). The platform never adds them to
its own safe return home.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `preflight.enabled` | `false` | Run the preflight check Skill before the run. | Per run; theatre, organization, global |
| `preflight.package-id` | `zqnt.system.flight-preparation` | Application that provides the preflight check Skill. | Per run |
| `preflight.capability-id` | `preflight` | Skill that performs the preflight check. | Per run |
| `preflight.minimum-battery-percent` | 0 | Refuse a run when the asset's battery is below this % (0 = no minimum). Only checked while `capability.safety.telemetry-check.enabled` is `true`. | Per run; theatre, organization, global |
| `preflight.maximum-wind-speed` | 0 | Refuse a run when the reported wind speed (m/s) is above this (0 = no limit). Only checked while `capability.safety.telemetry-check.enabled` is `true`. | Per run; theatre, organization, global |
| `flight.auto-takeoff.enabled` | `false` | Take off before the run. | Per run; theatre, organization, global |
| `flight.auto-takeoff.package-id` | `zqnt.system.flight-preparation` | Application that provides the auto-takeoff Skill. | Per run |
| `flight.auto-takeoff.capability-id` | `auto-takeoff` | Skill that performs the automatic takeoff. | Per run |
| `agent.human-approval.enabled` | `false` | Ask a person to approve before the run starts. | Per run; theatre, organization, global |
| `agent.human-approval.package-id` | `zqnt.system.flight-preparation` | Application that provides the approval Skill. | Per run |
| `agent.human-approval.capability-id` | `agent-approval` | Skill that asks for the approval. | Per run |
| `risk.human-approval.required` | `false` | Refuse a run in which a high-risk command (any `flight.*`, `navigation.*`, `dock.*` or `mission.*` command, or `asset.reboot`) can be reached without passing a Human Approval step first. | Per run; theatre, organization, global |

What the built-in steps do, as shipped:

- **preflight** sends the command `flight.preflight`. None of the current edge adapters or the simulator
  offers that command, so switching `preflight.enabled` on makes runs fail at that step. Point
  `preflight.package-id` and `preflight.capability-id` at a Skill of your own instead, or leave it off.
- **auto-takeoff** sends `flight.takeoff` without parameters, so the aircraft takes off in place to the
  default 40 m above the takeoff point. It is added in front of every run, so use it only where runs
  start with the aircraft on the ground.
- **agent-approval** is a Human Approval step named **AI agent proposal approval** with a 15-minute
  timeout. A run nobody approves within that time fails.

`risk.human-approval.required` adds no step of its own. With it on, a Skill must contain a Human Approval
node before its high-risk commands, or `agent.human-approval.enabled` must also be on. A single command
from Remote Control has no approval step, so it is refused.

### Return after a direct go-to

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `route.return.enabled` | `false` | After a direct go-to (a single `navigation.go_to` command, not a Skill), fly back to the dock. | Per run; theatre, organization, global |
| `route.return.finish-action` | `GO_HOME` | What the returning go-to ends with. Only `GO_HOME` adds anything: a leg back to the dock at the go-to's height, then a return to home. `NO_ACTION` and `AUTO_LANDING` add nothing. | Per run; theatre, organization, global |

### Safety checks before a run starts

These checks run when a run starts and again when it resumes. They are all off by default.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `capability.safety.telemetry-check.enabled` | `false` | Refuse a run when the asset's telemetry is older than `capability.safety.telemetry-max-age` or has no position. Also enables the battery and wind limits above. | Per run; theatre, organization, global |
| `capability.safety.capability-check.enabled` | `false` | Refuse a command the asset does not currently report as available. | Per run; theatre, organization, global |
| `capability.safety.schema-drift-check.enabled` | `false` | Refuse a run whose commands no longer match the schemas the asset reports. | Per run; theatre, organization, global |
| `capability.safety.telemetry-max-age` | 30000 | Milliseconds after which an asset's telemetry counts as stale for the telemetry check. | GLOBAL only |
| `capability.safety.capability-max-age` | 30000 | Milliseconds after which an asset's reported capabilities count as stale. | GLOBAL only |
| `capability.safety.check-timeout` | 5000 | Milliseconds a safety check may take before it counts as failed. | GLOBAL only |
| `capability.safety.command-timeout` | 10000 | Milliseconds a safety command (for example the return home after a failure) may take to be accepted. | GLOBAL only |

With `preflight.minimum-battery-percent` set, a run is also refused when the asset's telemetry carries no
battery percentage (**Battery state is missing**). With `preflight.maximum-wind-speed` set, a run is
refused when the asset reports no wind speed (**Wind speed is missing**); a dock reports wind, an
aircraft on its own usually does not.

### Internal

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `capability.execution.transition.max-retries` | 32 | How often a conflicting execution state write is retried. | GLOBAL only |
| `capability.execution.transition.retry-backoff` | 2 | Milliseconds before the first retry of a conflicting execution state write. | GLOBAL only |
| `capability.execution.transition.max-retry-backoff` | 100 | Upper limit in milliseconds for the retry backoff. | GLOBAL only |

Leave these alone unless Zequent support asks you to change them.

### Safety-relevant keys

A per-run override of any key starting with `risk.`, `agent.human-approval.`, `preflight.`,
`capability.safety.`, `route.nfz.`, `route.return.` or `flight.auto-takeoff.` switches safety behaviour
for that run. Only an organization admin or a system admin may send one. A request from anyone else is
refused with HTTP 403, and the error lists the keys that were not allowed. The same applies to the run
options `preflightProfile` and `nfzPolicyProfile`.

Organization and site entries on the Technical Config screen are already limited to admins.

## Dispatch Rules

Dispatch rules decide **which asset responds when a run doesn't name one**: a detection, an event
trigger set to choose by rule, a schedule without a fixed asset, an Integration Hub alarm, or an
Application that pins no asset. A run that names its asset never asks them. For the full order in which
an Application's own asset or site scope is tried first, see
[Which asset an execution runs on](applications-and-skills-2.0.md#which-asset-an-execution-runs-on).

### How they are checked

The rules form one list, checked **top to bottom**. The first rule that applies and can choose an asset
decides. A rule that does not apply to this run, or applies but cannot choose, hands over to the next.
The order is set in the console by dragging rules up or down. At the bottom sits the built-in fallback
**Any Available Asset Fallback**: everywhere, no conditions, available assets with at least 20 %
battery, most battery first. Keep it, or another rule without conditions, active; otherwise a schedule
or a manual run, which usually has no target position and no detected object, cannot get an asset.

Some things hold for every rule, whatever it says:

- Offline assets are never chosen, and neither are assets that report no battery level or no position.
- Assets of another organization are never chosen.
- Free assets come before busy ones. A busy asset is only taken over by a run of strictly higher
  priority, and the run that held it is cancelled.
- When the run belongs to a site, that site's own assets are asked first.

### What a rule says

A rule reads as a sentence, for example: *When a drone is detected at site North Range, send the nearest
available aircraft with at least 30 % battery within 5 km.* Its parts:

- **Where it applies** (scope): everywhere (system admins only), one organization, one site, or runs
  whose Skill needs one capability (system admins only).
- **When** (conditions, all must hold; none means every run in its scope):

  | Condition | Notes |
  | --- | --- |
  | The run has a target position | A detection always has one; a schedule or a manual run often has none. |
  | Detected object type | Only detections carry it. A rule with this condition never applies to a scheduled, manual or integration run. |
  | Detection confidence | Only detections carry it. |
  | Detected by asset (serial) | Only detections carry it. |
  | The run belongs to site | See [Which value a run uses](#which-value-a-run-uses) for how a run's site is found. |
  | The run needs capability | The capability the run's Skill needs. |

- **Which assets may answer** (constraints):

  | Constraint | API name | Notes |
  | --- | --- | --- |
  | Battery at least | `BATTERY_MIN` | Assets below this battery % are skipped. |
  | At most this far from the target | `DISTANCE_MAX` | In metres. Needs a target position; a run without one cannot be judged, and the next rule is asked. |
  | Only this kind of asset | `ASSET_TYPE_REQUIRED` | Aircraft or dock. |
  | Only assets stationed at site | `ASSET_IN_SITE` | Assets assigned to that site under **Deploy › Theatres**. |
  | Available | `AVAILABILITY` | Found in older rules. Every rule already skips unavailable assets. |

- **Which one is chosen** (strategy):

  | Strategy | API name | Notes |
  | --- | --- | --- |
  | Nearest to the target | `NEAREST` | Needs a target position. A run without one passes over the rule and the next one answers. |
  | Most battery | `MOST_BATTERY` | Works for every run. |

The console offers templates to start a rule from. In the API, every rule has the type
`DETECTION_RESPONSE`, whatever kind of run it answers.

### Test a rule: what would happen?

The test asks the live platform which asset would be chosen right now for a run like the one you
describe, and why, without starting, holding or recording anything. You describe the run: no target
position (a schedule or a manual run), aimed at a position, or a detection (object type, confidence,
position); optionally its site, its organization (system admins), and its priority. The answer shows:

- the asset that would be chosen and the rule that chose it, or why none could;
- for every rule, whether it chose, did not apply (with the scope or condition that kept it out), found
  no asset that passed (with each asset's reason), could not choose (for example nearest without a
  position), or was not reached;
- every asset considered, with its battery, its distance from the target and why it was not chosen.

The test is also available as `POST /api/admin-console/policies/simulate`. It is limited to your own
organization unless you are a system admin.

When a real run cannot get an asset, the platform gives the reason in the same terms: the run is refused,
or a schedule's firing is recorded as skipped, with the reason.

## Recipes

### Wider no-fly-zone margin for one site

Add a Technical Config entry: key `route.nfz.buffer_m`, value type `DOUBLE`, value `50`, scope `THEATRE`,
the site. Runs that belong to that site keep 50 m from every `HARD_BLOCK` zone; every other run keeps the
default 20 m. Runs created before the change keep their margin. A fly-to from Remote Control has no site,
so it uses the organization or global value; for an organization-wide margin, use scope `ORGANIZATION`.

### Earlier low-battery return for one organization

Add `route.safety_return.floor_percent` = `40` (`DOUBLE`) with scope `ORGANIZATION`. Every aircraft of
that organization in the air is sent home at 40 % at the latest. To keep more spare battery on long
routes, raise `route.safety_return.reserve_percent` instead: it is added on top of what the way home
actually needs.

### Refuse runs below a battery level

For one organization, add two `ORGANIZATION` entries: `capability.safety.telemetry-check.enabled` =
`true` (`BOOLEAN`) and `preflight.minimum-battery-percent` = `40` (`DOUBLE`). A run is then refused when it
starts if the asset's telemetry is stale, has no position, has no battery percentage, or shows less than
40 %. To pick only charged assets for runs that name none, use a dispatch rule with
**Battery at least** instead; it skips weak assets rather than refusing the run.

### Ask a person before every run in one site

Add `agent.human-approval.enabled` = `true` with scope `THEATRE`. Every run that belongs to that site
starts with an approval step (**AI agent proposal approval**, 15-minute timeout). Approve it from the
notification, the header's approval indicator or the execution page.

### Test the low-battery return in the simulator

1. Create a simulated aircraft and take it off **through the platform** (a takeoff command from Remote
   Control or a Skill), so the platform knows its home. See
   [Test with simulators](/docs/2.0/console/simulators).
2. Draw a `HARD_BLOCK` zone between the aircraft and its home, and fly the aircraft past it.
3. Add `route.safety_return.floor_percent` with scope `ORGANIZATION` for your organization and a value
   above the simulated aircraft's current battery, for example `95`.
4. Within a few seconds the platform sends the aircraft home around the zone, and the console shows the
   **Low-battery return home** alert.
5. Delete the entry (or switch it off) afterwards. The return triggers only once per flight; to test
   again, land the aircraft first.

Do not do this on a real aircraft in flight: it will be sent home.

### Send detections to the nearest drone, everything else to any drone

1. Add a rule for your organization with the condition **The run has a target position** = yes,
   **Battery at least** 30 % and **At most this far from the target** 5000 m, strategy **Nearest to the
   target**. Put it at the top.
2. Keep **Any Available Asset Fallback** at the bottom for runs without a position.
3. Use the test with "a detection" and with "no target position" to see each run answered by the rule
   you expect.

## See also

- [No-fly zones and safe returns](airspace-safety-2.0.md)
- [Applications & Skills](applications-and-skills-2.0.md)
- [Deploy and automate](/docs/2.0/console/deploy-and-automate)
