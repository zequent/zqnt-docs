# No-fly zones and safe returns

This page explains how Zequent 2.0 keeps aircraft out of your no-fly zones: where zones are stored, what
happens to each kind of flight command that would cross one, how heights are measured, and how the
platform brings an aircraft home on low battery before its own firmware does. The settings named here
are Technical Config keys; see [Technical configuration and dispatch rules](configuration-2.0.md) for
how to change them for one organization or one site.

## Where zones live and who checks them

- **Drawn in the console.** Zones are drawn in **Operate → Remote Control** with the **Zone** draw mode
  and managed under **Plan**. See [Draw a no-fly zone](/docs/2.0/console/operate#draw-a-no-fly-zone).
- **Stored per organization.** Every zone belongs to one organization and is stored by the Connector
  service in the platform database. Every flight of that organization respects every active zone of
  that organization. A zone can be filed under a site (theatre); it still applies to every flight of
  the organization.
- **Checked centrally.** Mission Autonomy checks every movement command right before it is sent to a
  device. Edge adapters never see zones and never reroute: they fly exactly what they are sent, and
  what they are sent has already been checked.

This applies wherever the command comes from: a Skill node, a node of a composed Skill, a scheduled or
triggered run, or a command sent directly from Remote Control.

With preflight switched on (`preflight.enabled`), a run is also checked before it starts: it fails its
preflight step when the takeoff point or a known target lies inside an active `HARD_BLOCK` zone. See
[Flight preparation](configuration-2.0.md#flight-preparation).

If the zones cannot be loaded (for example while the Connector service is unreachable), every movement
command is refused, except a return home. A return home is sent unchecked, with a warning on the step,
because refusing it would leave the aircraft where it is.

### What a zone has

| Property | Meaning |
| --- | --- |
| Shape | A polygon of at least three points. |
| Enforcement | `HARD_BLOCK`, `REQUIRE_APPROVAL` or `ADVISORY`. The console shows them as **Block**, **Approval** and **Advisory**. A new zone is `HARD_BLOCK`. |
| Active | Inactive zones are kept but ignored, by the platform and by the console's proximity alarm. The map draws them grey, marked **(not enforced)**. A new zone is active. |
| Altitude band | Optional lower and upper edge in metres above the takeoff point (**From height (m)** and **Up to height (m)** in the console). Blank means from the ground, or no ceiling. |

All of these are set in the console, in the zone list under **Plan** in Remote Control. Through the Admin
Console API they are `PUT /api/admin-console/no-fly-zones/{id}` with `enforcement`, `active`,
`minAltitudeMeters` and `maxAltitudeMeters`.

A zone's altitude band only exempts a flight whose heights are all known to lie outside the band. When
a height is unknown, the zone applies.

### Enforcement levels

| Enforcement | What the platform does with a flight path that crosses the zone |
| --- | --- |
| `HARD_BLOCK` | Never flown through. The platform flies a detour around it. The command is refused only when no detour exists, for example when the target lies inside the zone. |
| `REQUIRE_APPROVAL` | A Human Approval step is inserted in front of the flight. The aircraft only moves after a person approves. Rejecting it takes the step's failure branch. |
| `ADVISORY` | Flown as commanded. The step records a warning. |

An aircraft that is already inside a `HARD_BLOCK` zone may always fly out of it. The zone is left out of
the planning for that flight, and a warning says so.

## The safety margin

A detour keeps a margin around every `HARD_BLOCK` zone: the Technical Config key `route.nfz.buffer_m`,
in metres, default **20**.

- Refusal is judged against the real zone, not the margin. A target 5 m outside a zone is allowed.
- The detour's turn points lie at least the margin away from the zone. A start or target that lies
  inside the margin is reached the shortest way.
- When zones lie so close together that the margin cannot be kept, the leg is planned against the real
  zones and the step records a warning.
- A negative or unreadable value falls back to 20 m. Values above 1000 m are capped at 1000 m.

## What happens per command

The platform checks the path from where the aircraft is now (its telemetry) to where the command sends
it.

### `navigation.go_to`

The same rules apply to a go-to node in a Skill and to a fly-to sent from Remote Control.

- **Through a `HARD_BLOCK` zone:** the platform inserts extra go-to legs to the detour's turn points in
  front of the go-to. The aircraft flies around the zone and then on to the target.
- **No detour exists** (for example, the target is inside the zone): the command is refused with the
  reason. In a Skill, the step fails and its failure branch runs.
- **Through a `REQUIRE_APPROVAL` zone:** a person approves first.
- **Through an `ADVISORY` zone:** the aircraft flies, with a warning.

#### Fly-to from Remote Control

Remote Control asks the platform about the path before it sends a fly-to, and shows what will happen:

| The path | What the operator sees |
| --- | --- |
| Clear | Nothing. The fly-to is sent. |
| Through a `HARD_BLOCK` zone, detour exists | A dialog naming the zone and the number of turn points, with **Fly detour**. An organization admin also gets **Override: fly straight**. |
| Through a `HARD_BLOCK` zone, no detour | **Refused**, with the reason. An organization admin can override it. |
| Through a `REQUIRE_APPROVAL` zone | **Send for approval**: the flight waits until someone approves it. An organization admin can fly it straight away with an override instead. |
| Through an `ADVISORY` zone | A warning with **Fly anyway**. |

**The override.** Only an organization admin or a system admin may override a zone, and only on a direct
command from Remote Control, never on a Skill's step. An overridden fly-to flies the straight line. The
override is recorded in the audit log with who asked for it, and the step carries a warning.

If the console cannot reach the check, it sends the fly-to anyway; the platform checks it again before
dispatch and reroutes or refuses it.

### `flight.return_to_home`

The aircraft flies its own return home, straight. When that straight line crosses a `HARD_BLOCK` zone, the
platform inserts go-to legs to the detour's turn points first, at the return height. The aircraft then
starts its own return from the last turn point, with home in plain sight.

A return home that the platform starts on its own (the low-battery return below, or the automatic return
after a failed or cancelled flight step) is never refused and never waits for a person:

- it flies the detour when one exists;
- it flies through a `REQUIRE_APPROVAL` zone without asking, and says so on the step;
- when no detour exists, it is still sent, as the aircraft's own straight return, with a warning
  saying which zone it crosses. Leaving the aircraft hovering where it is is never the safer choice.

When the aircraft's home or its current position is unknown, the return is sent unchecked, with a
warning. When a detour leg of a platform return fails, the aircraft is sent its own straight return
instead, with a warning.

### `mission.waypoint.execute`

- Detour turn points are inserted as extra waypoints between the mission's own waypoints. Each inserted
  point takes the height and speed of the waypoint after it.
- **The mission's own return home** after its last waypoint is routed too. DJI flies that return unless
  the mission's `waylineFinishAction` says otherwise; MAVLink (PX4/ArduPilot) and the simulator always
  return to their launch point. The device flies that leg on its own, so the platform appends the
  detour's turn points as extra waypoints at the end of the mission. The device's return then starts
  from the last of them.
- When no detour exists for the mission's return, the mission still flies, and the step carries a
  warning naming the zone the return crosses.

### Manual control

Manual control (keyboard or gamepad) is never blocked by a zone. Remote Control warns the pilot instead:
within 10 m of an active zone it shows how far away the zone is, and inside a zone it shows
**Restricted airspace**, pulses red and sounds an alarm unless sounds are muted.

## Home position

A return home is checked toward the aircraft's home:

- **An aircraft in a dock (DJI):** the dock's position.
- **An aircraft without a dock (the simulator, MAVLink):** the point it last took off from **through
  the platform**. The platform notes the aircraft's position when it sends a `flight.takeoff`, and only
  when the aircraft reports its height above the takeoff point and is on the ground (2 m or lower). A
  takeoff sent while the aircraft is already flying, or while it reports no height, does not change
  the home. The point is forgotten 24 hours after the last takeoff through the platform.

An aircraft that took off outside the platform (for example with its own remote) has no known home
until its next takeoff through the platform. Its return home is then sent unchecked, with a warning, and
the low-battery return only alerts (see below).

## Altitudes

Every `altitude` on `flight.takeoff` and `navigation.go_to` is a height **in metres above the takeoff
point**, on DJI, MAVLink and the simulator alike. It is a target height, not a climb: asking for 30 m
while flying at 20 m climbs 10 m, asking again holds 30 m. A go-to node can therefore run again and
again without the aircraft creeping upward.

- An omitted altitude means **40 m** above the takeoff point.
- A fly-to from the Remote Control map keeps the aircraft's current height above the takeoff point. When
  that height is unknown, or the aircraft is still on the ground, the altitude is left out (40 m).
- Zone altitude bands use the same reference.
- The SAPIENT adapter converts this height into SAPIENT's own height reference itself.

## Low battery: the platform's safe return

Every aircraft has a low-battery return of its own (DJI's smart return-to-home, PX4's low and critical
battery failsafes, the simulator's). All of them fly straight home, through any no-fly zone in the way.
The platform knows the zones, so it acts first.

### How it decides

About every 5 seconds, for every aircraft that is in the air (more than 2 m above its takeoff point, with
telemetry no older than 30 seconds), the platform works out how much battery the **zone-safe** way home
needs:

- **needed** = (length of the path home around the `HARD_BLOCK` zones ÷ speed + current height ÷ descent
  speed) × drain rate + landing allowance
- The speed is `route.safety_return.cruise_speed_mps`, or the aircraft's measured ground speed when that
  is slower.
- The drain rate is learnt from the aircraft's own battery during this flight (at least a 2 % drop over
  at least 2 minutes, within the last 5 minutes). Until then it is
  `route.safety_return.drain_percent_per_minute`.

It sends the aircraft home when the battery is at or below the **largest** of:

- the floor, `route.safety_return.floor_percent` (default 30 %);
- needed + `route.safety_return.reserve_percent` (default 10 %);
- for DJI, the return level the aircraft itself reports + `route.safety_return.firmware_margin_percent`
  (default 5 %).

The default floor of 30 % lies above PX4's default low and critical battery levels (15 % and 7 %).

### What it does

- **Starts a return home.** The return flies the detour around the zones, as described for
  `flight.return_to_home` above.
- **Takes the aircraft over** from any run on it, whatever that run's priority. The run that was using
  the aircraft is cancelled, and no other run can take the aircraft back.
- **Ignores the organization's run preparation.** The return runs without preflight, automatic takeoff,
  approval steps or start-time safety checks, because the low battery is the reason it runs.
- **Once per flight.** It does not trigger again until the aircraft has been back on the ground. It never
  triggers for an aircraft that is already returning or landing.
- **Alerts.** A **safety alert** appears in the console's notifications and Live feed, with the battery,
  the threshold it acted at and the length of the way home. It links to the return run, or to the asset
  when no run was started. When an operator has to act (manual control, no way home, no route around a
  zone, a return that did not start), it also plays a sound and stays on screen until dismissed. Keep
  **Safety alert** switched on under **Alerting & Rules**. Past alerts of an asset are listed on its
  **Safety** tab.

It only alerts, and does not move the aircraft, when:

- **someone has manual control.** The platform never takes the sticks from an operator. The alert
  (**Low battery: return home now**) repeats every 2 minutes.
- **the home is unknown.** There is no way home to plan. The alert is **Low battery: home position
  unknown**.

When no route around the zones exists, the return is still sent and the alert says **NO SAFE ROUTE**:
the aircraft flies its own straight return through the zone. Watch it on its way back. If the return
cannot be started at all, the alert says so and the platform tries again on its next check.

### Settings

| Key | Default | Accepted values |
| --- | --- | --- |
| `route.safety_return.enabled` | `true` | `true` / `false` |
| `route.safety_return.floor_percent` | 30 | 0 – 100 |
| `route.safety_return.reserve_percent` | 10 | 0 – 100 |
| `route.safety_return.cruise_speed_mps` | 10 | 0.5 – 50 |
| `route.safety_return.drain_percent_per_minute` | 2.5 | 0.01 – 50 |
| `route.safety_return.landing_allowance_percent` | 3 | 0 – 100 |
| `route.safety_return.descent_speed_mps` | 3 | 0.1 – 20 |
| `route.safety_return.firmware_margin_percent` | 5 | 0 – 100 |

A value outside its range is ignored and the default is used. These keys are looked up for the
aircraft on every check: an entry for the asset itself, then the site (theatre) it is stationed at,
then its organization, then GLOBAL. A change applies to aircraft already in the air.

The check itself can be switched off for a whole installation with the Mission Autonomy environment
variable `ZQNT_SAFETY_RETURN_ENABLED=false`; how often it runs is `ZQNT_SAFETY_RETURN_EVERY` (default
`5s`).

## Limits

- **Link loss.** When the platform cannot reach the aircraft, its commands cannot arrive. The aircraft's
  firmware decides what happens, and its own return flies straight.
- **Firmware thresholds.** Keep the aircraft's own low-battery thresholds below the platform's floor, so
  that the platform acts first. When the firmware acts first, its return flies straight.
- **The aircraft's own automatic returns** (for example a firmware failsafe, or a DJI return started
  outside the platform) cannot be rerouted. Only returns the platform sends are routed around zones.
- **Manual control** is never blocked. The pilot is warned, not stopped.

## See also

- [Technical configuration and dispatch rules](configuration-2.0.md)
- [Applications & Skills](applications-and-skills-2.0.md)
- [Operate your fleet: Remote Control](/docs/2.0/console/operate#remote-control)
