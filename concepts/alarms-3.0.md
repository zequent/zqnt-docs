# Alarm lifecycle (3.0 preview)

An **alarm** is something the operators of a site should react to: an object the sensors detected
and nobody has cleared, a warning an asset raised about itself, or an event trigger that fired and
could not start its run. The platform opens alarms, keeps them current while the object is reported,
starts the automatic reaction set up for the site, and closes them again when the target is gone —
so the inbox holds what is happening now, not what happened last night.

This page describes a preview; details can change until 3.0 is released.

## One alarm per target

A detected object is one alarm, however many sensors see it. The platform groups the reports of its
sensors into **tracks** (one track per physical object — a radar, a camera and a Remote ID receiver
reporting the same drone feed one track), and the alarm is keyed by that track:

- A report of a track with an open alarm **updates** that alarm: when it was last seen, where it is,
  what it is, and which sensors see it. It never opens a second one — not from another sensor, and
  not from a sensor at another site.
- The alarm belongs to the **site it was first reported at**. Its card lists every sensor that has
  reported the track since, so an operator at the border theatre sees "seen by BRD-RADAR-C,
  BRD-RADAR-E, BRD-RADAR-W" on one card instead of three cards.
- Reports without a track (a sensor that does not identify its objects) are keyed by sensor and
  object id, per site, as before.

Asset warnings stay one alarm per asset and code; trigger alarms one per trigger.

When the platform is upgraded to this version, open duplicates of one track are merged: the oldest
alarm stays, the others move their notes and runs into it and are resolved as duplicates ("Merged
into …").

## Expiry: the target is lost

An alarm whose track is no longer reported expires on its own. Each site sets **how long** after
the track's last report (default **15 minutes**, *Alarm rules → Resolve an alarm whose target is
not reported for*):

| The alarm is … | After the target-lost time |
| --- | --- |
| nobody's (not taken, not handed to a role or shift group) | resolved as **Target lost** and leaves the inbox |
| taken by an operator, or handed to a role or shift group | kept, with a **Target lost** step on its timeline; the operator resolves it |

Either way the "No progress" flag stops at that moment: there is nothing left to make progress on.
The inbox shows "Target lost 12 min ago" instead. Should the track be reported again, the alarm is
back to normal (a resolved one reopens within the site's reopen window, as any resolved alarm does).

An alarm is also closed when an operator **classifies its track** as a friend or neutral (on the
track, in the console's detections view): the alarm is resolved as a false alarm by that operator,
whoever holds it and whatever is reacting to it. A track classified as suspect or hostile raises the
alarm's severity with its next report.

The "No progress for …" flag itself now counts in minutes and hours ("No progress for 23 h"), never
in thousands of minutes.

## Automatic reactions and retries

An event trigger that answers detections at the site starts an Application for the alarm — the
**automatic reaction** — and the run is linked to the alarm. There is **one automatic reaction per
alarm**:

- While the run is going, further reports of the track start nothing.
- Once it **succeeded** (or was cancelled by a person), further reports start nothing either; the
  operator decides what happens next and can start another reaction from the alarm card.
- When it **fails**, the platform tries again — but not on the next report, and not forever. The
  site sets how many **retries** (default **3**) and the **wait before each retry** (default
  **1, 5, 15 minutes**; the last value repeats when there are more retries than waits). After the
  wait, the track's next report starts the next attempt. Every attempt is a step on the alarm's
  timeline, with the run's own reason for failing ("no asset became free", "target out of coverage",
  …) and the fix-it actions the run view offers.
- When the last allowed retry fails too, the alarm says so **once** — "Reaction failed 4 times,
  needs an operator" — and no automatic run starts for it any more. It expires like any other alarm
  when its target is gone; an operator can still take it, start a reaction by hand, or resolve it.

A run an operator starts from the alarm is linked like an automatic one but is never retried by the
platform.

The settings live with the site's other alarm rules (escalation, stall, reopen) in the *Operate*
screen's duty panel and in `AlarmSiteSettings` of the `zqnt.alarm.v3` API:

| Setting | Default | Meaning |
| --- | --- | --- |
| `target_lost_after_minutes` | 15 | A detection alarm whose track is not reported for this long is resolved as Target lost, or noted so when someone holds it |
| `reaction_retries` | 3 | How often a failed automatic reaction is tried again before the alarm needs an operator |
| `reaction_retry_backoff_minutes` | 1, 5, 15 | The wait before each retry; the last value repeats |

## The inbox

The inbox lists the open alarms of the chosen site (every site on the overview), **newest first**:
the alarm still being reported stays on top; an unassigned critical alarm is always on top. Each
card is one target, with how many sensors see it and how many reports it stands for. Past 20 cards
the rest fold behind "N more"; the counts in the header are the open alarms, folded or not. The
keyboard flow is unchanged: **A** next alarm, **Enter** take, **1–3** start a reaction, **H** hand
over, **R** resolve, **Esc** back.

## What an alarm carries

For an integration reading alarms (`ListAlarms`, `GetAlarm`, or the operational event channel, kind
`ALARM`):

| Field | Meaning |
| --- | --- |
| `track_id`, `sensor_sns`, `source_sn` | The track, every sensor that reported it (first to latest), and the latest of them |
| `target_lost_at` | Set once the track has not been reported for the site's target-lost time; cleared when it is reported again |
| `resolution = TARGET_LOST` | The alarm expired; `resolution_note` says after how long |
| `reaction.attempts` | Automatic runs started for the alarm so far |
| `reaction.next_attempt_at` | After a failure: when the next attempt may start |
| `reaction.needs_operator` | The last allowed retry failed; no automatic run starts any more |
| Timeline steps `TARGET_LOST`, `NEEDS_OPERATOR`, `REACTION_FAILED` | The expiry note on a held alarm, the hand-over to an operator, each failed attempt with its reason |
