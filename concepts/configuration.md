# Technical configuration and dispatch rules

Zequent 2.0 has two places where an organization tunes how the platform behaves, both under
**Configuration** in the Admin Console:

- **Technical Config**: named settings (keys) such as the no-fly-zone safety margin or the low-battery
  floor, set platform-wide, per organization, per site or per asset.
- **Dispatch Rules**: which asset responds when a run does not name one. Older console screens and the
  Admin Console API call them **operational policies** (`/api/admin-console/policies`).

This page explains how both work, lists every runtime key with its default, and ends with recipes.

## Who can change what

| | Org admin | System admin |
| --- | --- | --- |
| See the Configuration screen | Yes | Yes |
| Technical Config entries | Only `ORGANIZATION` (their own organization), `THEATRE` (a site of their own organization) and `ASSET` (an asset of their own organization) | Every scope |
| Dispatch rules | Only for their own organization or one of its sites | Every scope |

Every entry that could apply to you is visible, including platform-wide ones you cannot edit. Operators
(users without an admin role) cannot open the screen.

## Technical Config

Each entry has a **Key**, a **Value** with a value type (`STRING`, `INTEGER`, `LONG`, `BOOLEAN`,
`DOUBLE` or `JSON`), a **Scope**, an optional scope target and an **Active** switch. An inactive entry is
ignored.

The console offers only the scopes at which the key you type actually takes effect:

| Key | Scopes offered |
| --- | --- |
| Keys a run uses (`route.*`, `preflight.*`, `flight.auto-takeoff.*`, `agent.human-approval.*`, `risk.human-approval.required`, the `capability.safety.*.enabled` switches, and any key a Skill sets a default for) | `GLOBAL`, `ORGANIZATION`, `THEATRE`, `ASSET` |
| Platform timing keys (the `capability.safety.*` timeouts and maximum ages, `capability.execution.transition.*`) | `GLOBAL` only |
| The AI adapter's `ai.*` keys | `GLOBAL`, and `ADAPTER` with the target `edge-ai` (every AI adapter) or one adapter's serial number |

For `ASSET`, you pick the asset by name; the entry stores its serial number. `SERVICE` and `ASSET_TYPE`
are no longer offered, because nothing reads them. Existing entries at those scopes stay visible and
editable, marked **no effect at this scope**.

Every runtime key below exists as a `GLOBAL` entry from the start, holding the value the platform uses
when nothing else is set, so you can see it and override it.

### Which value a run uses

The settings a run uses are worked out once, when the run is created, in this order (first match wins):

1. **A per-run override** sent with the run request (`configOverrides` in the run's options). Safety
   settings may only be overridden by an organization admin or a system admin; see
   [Safety-relevant keys](#safety-relevant-keys).
2. **`ASSET`**: an entry for the asset the run uses.
3. **`THEATRE`**: an entry for the site the run belongs to.
4. **`ORGANIZATION`**: an entry for the run's organization.
5. **The Skill's own default**: a default the Skill (or its Application) sets for the key.
6. **`GLOBAL`**.

The editor sums this up as **Most specific wins: asset › site › organization › global**. A `GLOBAL`
value never overrides a default the Skill author chose; an asset, site or organization entry does. The
**Scope resolution chain** on the Technical Config screen lists the entries for a key by scope; it does
not show a Skill's own default.

**The site a run belongs to** is the site named by whatever started it (for example a site-bound
trigger); otherwise the site its Application is deployed to; otherwise the site whose area contains the
run's target position (`latitude`/`longitude` in its inputs). A run with none of these, such as most
commands sent from Remote Control, has no site, and only asset, organization and global entries apply.

Some keys are not part of a run's settings and are read live instead:

- the low-battery return keys `route.safety_return.*`: per aircraft, on every check, from the asset,
  then the site it is stationed at, then its organization, then `GLOBAL`;
- the margin Remote Control uses when it checks a fly-to before sending it: asset, then organization,
  then `GLOBAL`;
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

See [No-fly zones and safe returns](airspace-safety.md) for what these do.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `route.nfz.buffer_m` | 20 | Safety margin in metres every flight keeps around a `HARD_BLOCK` no-fly zone; detours are planned this far outside it. Capped at 1000. | Per run; asset, theatre, organization, global |
| `route.safety_return.enabled` | `true` | Send an airborne aircraft home around the no-fly zones on low battery, before its firmware flies its own straight return. | Live; asset, theatre, organization, global |
| `route.safety_return.floor_percent` | 30 | Battery % at or below which the platform always sends the aircraft home. Keep firmware low-battery thresholds below this. | Live; asset, theatre, organization, global |
| `route.safety_return.reserve_percent` | 10 | Battery % added on top of what the path home around the zones needs. | Live; asset, theatre, organization, global |
| `route.safety_return.cruise_speed_mps` | 10 | Assumed speed home in m/s; the measured ground speed is used when slower. | Live; asset, theatre, organization, global |
| `route.safety_return.drain_percent_per_minute` | 2.5 | Assumed battery drain in %/min until the aircraft's own drain has been measured in flight. | Live; asset, theatre, organization, global |
| `route.safety_return.landing_allowance_percent` | 3 | Battery % allowed for the landing at home. | Live; asset, theatre, organization, global |
| `route.safety_return.descent_speed_mps` | 3 | Assumed descent speed in m/s for the height to lose before landing. | Live; asset, theatre, organization, global |
| `route.safety_return.firmware_margin_percent` | 5 | Act this many % above the return-home battery level the aircraft reports (DJI). | Live; asset, theatre, organization, global |

For the safe-return keys, "theatre" means the site the asset is stationed at, not the site of a run.

### Flight preparation

When switched on, these put extra steps in front of runs created in that scope. The steps come from the
built-in Application `zqnt.system.flight-preparation` ("Zequent system flight preparation"). The
platform never adds them to its own safe return home.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `preflight.enabled` | `false` | Before a run that sends an aircraft out (takeoff, fly-to, waypoint mission) or that gets an automatic takeoff, run the preflight check. | Per run; asset, theatre, organization, global |
| `preflight.package-id` | `zqnt.system.flight-preparation` | Application that provides the preflight check Skill. | Per run |
| `preflight.capability-id` | `preflight` | Skill that performs the preflight check. | Per run |
| `preflight.minimum-battery-percent` | 0 | Refuse to start a run that sends an aircraft out when its battery is below this %, or its telemetry is missing or stale (0 = no minimum). See [Launch limits](#launch-limits). | Per run; asset, theatre, organization, global |
| `preflight.maximum-wind-speed` | 0 | Refuse to start a run that sends an aircraft out when the wind speed the asset reports (m/s) is above this (0 = no limit). See [Launch limits](#launch-limits). | Per run; asset, theatre, organization, global |
| `flight.auto-takeoff.enabled` | `false` | Take off before the run. | Per run; asset, theatre, organization, global |
| `flight.auto-takeoff.package-id` | `zqnt.system.flight-preparation` | Application that provides the auto-takeoff Skill. | Per run |
| `flight.auto-takeoff.capability-id` | `auto-takeoff` | Skill that performs the automatic takeoff. | Per run |
| `agent.human-approval.enabled` | `false` | Ask a person to approve before the run starts. | Per run; asset, theatre, organization, global |
| `agent.human-approval.package-id` | `zqnt.system.flight-preparation` | Application that provides the approval Skill. | Per run |
| `agent.human-approval.capability-id` | `agent-approval` | Skill that asks for the approval. | Per run |
| `risk.human-approval.required` | `false` | Refuse a Skill, Application, trigger, schedule or AI-agent run in which a high-risk command (any `flight.*`, `navigation.*`, `dock.*` or `mission.*` command, or `asset.reboot`) can be reached without passing a Human Approval step first. Direct commands from Remote Control and the platform's own safety return are exempt. | Per run; asset, theatre, organization, global |

What the built-in steps do:

- **preflight** is checked by the platform itself, the same way for every adapter and the simulator.
  The run shows it as a **Preflight** step. It checks, and reports every failure together:
  - the asset is online: it sends telemetry and does not report its aircraft as off. A DJI dock may
    report the aircraft sleeping inside it as off, which would fail this check; this has not been
    verified on hardware yet;
  - its telemetry is no older than `capability.safety.telemetry-max-age`;
  - the battery and wind limits, when set (see [Launch limits](#launch-limits));
  - every movement command of the run is reported **available** by the asset;
  - neither the takeoff point (for a run that takes off) nor any target of the run lies inside an active
    `HARD_BLOCK` no-fly zone. If the zones cannot be loaded, the check fails. A target that comes from an
    earlier step's output is not known yet; it is checked when that step flies.

  A failed check fails the step with the reasons, and the step's failure branch runs. Nothing is sent
  home for it. If the asset offers a preflight of its own (it reports `flight.preflight` as available),
  that runs after the platform's checks pass.

  The preflight step is only added in front of a run that sends an aircraft out (a takeoff, a fly-to or a
  waypoint mission) or that gets an automatic takeoff. A camera, gimbal or dock command, or a return
  home, gets no preflight.
- **auto-takeoff** sends `flight.takeoff` without parameters, so the aircraft takes off in place to the
  default 40 m above the takeoff point. It is added in front of every run created in its scope, so use it
  only where runs start with the aircraft on the ground.
- **agent-approval** is a Human Approval step named **AI agent proposal approval** with a 15-minute
  timeout. A run nobody approves within that time fails.

`risk.human-approval.required` adds no step of its own. With it on, a Skill must contain a Human Approval
node before its high-risk commands, or `agent.human-approval.enabled` must also be on (that inserts the
approval step). A command an operator sends directly from Remote Control is not affected: the operator
is already deciding. The platform's own safety return is never held up by it.

### Launch limits

`preflight.minimum-battery-percent` and `preflight.maximum-wind-speed` apply whenever they are above 0,
whether or not preflight or the telemetry check is switched on. They are checked when a run starts:

- only for a run that sends an aircraft out (a takeoff, a fly-to or a waypoint mission), and only before
  anything in it has started. A return home, a landing or a camera command is never refused, and a run
  that is already under way is not stopped half-way;
- with a limit set, a run is refused when the asset's telemetry is missing or stale;
- wind is only checked when the asset reports a wind speed (a dock does; an aircraft on its own usually
  does not);
- the platform's own safety returns are exempt.

The preflight step, when switched on, applies the same limits.

### Return after a direct go-to

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `route.return.enabled` | `false` | After a direct go-to (a single `navigation.go_to` command, not a Skill), fly back to the dock. | Per run; asset, theatre, organization, global |
| `route.return.finish-action` | `GO_HOME` | What the returning go-to ends with: `GO_HOME` flies back to above the dock at the go-to's height, then returns home and lands on the dock. `NO_ACTION` flies back to above the dock at the go-to's height and hovers there. `AUTO_LANDING` is refused when the run is created: the platform has no land command that works on every device. | Per run; asset, theatre, organization, global |

### Safety checks before a run starts

These checks run when a run starts and again when it resumes. They are all off by default.

| Key | Default | What it does | Scopes |
| --- | --- | --- | --- |
| `capability.safety.telemetry-check.enabled` | `false` | Refuse a run when the asset's telemetry is older than `capability.safety.telemetry-max-age` or has no position. | Per run; asset, theatre, organization, global |
| `capability.safety.capability-check.enabled` | `false` | Refuse a command the asset does not currently report as available. | Per run; asset, theatre, organization, global |
| `capability.safety.schema-drift-check.enabled` | `false` | Refuse a run whose commands no longer match the schemas the asset reports. | Per run; asset, theatre, organization, global |
| `capability.safety.telemetry-max-age` | 30000 | Milliseconds after which an asset's telemetry counts as stale for the telemetry check. | GLOBAL only |
| `capability.safety.capability-max-age` | 30000 | Milliseconds after which an asset's reported capabilities count as stale. | GLOBAL only |
| `capability.safety.check-timeout` | 5000 | Milliseconds a safety check may take before it counts as failed. | GLOBAL only |
| `capability.safety.command-timeout` | 10000 | Milliseconds a safety command (for example the return home after a failure) may take to be accepted. | GLOBAL only |

The battery and wind limits are not part of these switches; see [Launch limits](#launch-limits).

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

Organization, site and asset entries on the Technical Config screen are already limited to admins.

## Dispatch Rules

Dispatch rules decide **which asset responds when a run doesn't name one**: a detection, an event
trigger set to choose by rule, a schedule without a fixed asset, an Integration Hub alarm, or an
Application that pins no asset. A run that names its asset never asks them. For the full order in which
an Application's own asset or site scope is tried first, see
[Which asset an execution runs on](applications-and-skills.md#which-asset-an-execution-runs-on).

### How they are checked

The rules form one list, **checked top to bottom; the first rule that applies decides. If it finds no
suitable asset, the next rule is asked.** A rule that does not apply to this run hands over too. Move a
rule by dragging it, or with its up and down buttons; a new rule is added at the top. An org admin can
only move the rules they may edit; the others stay in place.

At the bottom sits the built-in fallback **Any Available Asset Fallback**, marked **Fallback**: everywhere,
no conditions, the available asset with the most battery and at least 20 %. It always stays last and
answers whatever no rule above could. Keep it, or another rule without conditions, active; otherwise a
schedule or a manual run, which usually has no target position and no detected object, cannot get an
asset.

A new installation also has three rules for detections above the fallback, in this order: **Nearest
asset for drone detections**, **Nearest aircraft for person detections** and **Most battery for drone
detections**. Each rule's description in the console says what it does. (Installations upgraded from an
earlier version get these names only where the rules were never changed.)

Wherever the console leaves the asset open (the run dialog, an Application's deployment, schedules and
event triggers), it says **Asset chosen by Dispatch Rules** with a link to this list.

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
  | Only assets stationed at site | `ASSET_IN_SITE` | Assets assigned to that site under **Manage › Theatres**. |
  | Available | `AVAILABILITY` | Found in older rules. Every rule already skips unavailable assets. |

- **Which one is chosen** (strategy):

  | Strategy | API name | Notes |
  | --- | --- | --- |
  | Nearest to the target | `NEAREST` | Needs a target position. A run without one passes over the rule and the next one answers. |
  | Most battery | `MOST_BATTERY` | Works for every run. |

**New rule** offers templates to start from: **Nearest available drone**, **Drone with the most
battery**, **Only drones stationed at a site**, **Respond to detections of a type** and **Blank**. The
editor shows the rule's sentence as you build it, and warns you when the rule would never be reached
(a rule above it has no conditions, covers the same runs and accepts every asset this one would) or when
a nearest rule lets runs without a target position fall through. In the API, every rule has the type
`DETECTION_RESPONSE`, whatever kind of run it answers.

### What would happen?

The **What would happen?** view asks the live platform which asset would be chosen right now for a run like the one you
describe, and why, without starting, holding or recording anything. You describe the run: no target
position (a schedule or a manual run), aimed at a position, or a detection (object type, confidence,
position); optionally its site, its organization (system admins), and its priority. The answer shows:

- the asset that would be chosen and the rule that chose it (with its sentence), or why none could;
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

### Refuse launches below a battery level

Add `preflight.minimum-battery-percent` = `40` (`DOUBLE`) with scope `ORGANIZATION`. A run of that
organization that sends an aircraft out is then refused when it starts if the asset's telemetry is
missing or stale or its battery is below 40 %. Returns home are never refused. To pick only charged
assets for runs that name none, use a dispatch rule with **Battery at least** instead; it skips weak
assets rather than refusing the run.

### Require a preflight for every flight of an organization

Add `preflight.enabled` = `true` (`BOOLEAN`) with scope `ORGANIZATION`. Every run of that organization
that sends an aircraft out starts with the **Preflight** step described in
[Flight preparation](#flight-preparation). Combine it with the battery and wind limits to set what the
preflight demands.

### A different margin or battery floor for one aircraft

Pick scope `ASSET` and the aircraft, for example `route.nfz.buffer_m` = `50` for a large aircraft, or
`route.safety_return.floor_percent` = `40` for one with an older battery. An asset entry beats site,
organization and global entries.

### Ask a person before every run in one site

Add `agent.human-approval.enabled` = `true` with scope `THEATRE`. Every run that belongs to that site starts
with an approval step (**AI agent proposal approval**, 15-minute timeout). Approve it from the
notification, the header's approval indicator or the execution page.

### Test the low-battery return in the simulator

1. Create a simulated aircraft and take it off **through the platform** (a takeoff command from Remote
   Control or a Skill), so the platform knows its home. See
   [Practice with simulated devices](/guides/simulators).
2. Draw a `HARD_BLOCK` zone between the aircraft and its home, and fly the aircraft past it.
3. Add `route.safety_return.floor_percent` with scope `ORGANIZATION` for your organization and a value
   above the simulated aircraft's current battery, for example `95`.
4. Within a few seconds the platform sends the aircraft home around the zone, and the console shows the
   **Low-battery return home** alert.
5. Delete the entry (or switch it off) afterwards. The return triggers only once per flight; to test
   again, land the aircraft first.

Do not do this on a real aircraft in flight: it will be sent home.

### Send detections to the nearest drone, everything else to any drone

Add a **Nearest available drone** rule above **Any Available Asset Fallback**, and keep the fallback
for runs without a position. Step by step, with the **What would happen?** test:
[Choose which drone responds](/guides/dispatch).

## See also

- [No-fly zones and safe returns](airspace-safety.md)
- [Applications & Skills](applications-and-skills.md)
- [Choose which drone responds](/guides/dispatch)
- [Start a response when something happens](/guides/events)
