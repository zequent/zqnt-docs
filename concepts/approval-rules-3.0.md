# Approval rules (3.0 preview)

Some commands should only reach a device after a person has said yes. On a 3.0 platform this is
decided by **approval rules**: one matrix per installation, organization and site that says, for
every risk level and for whoever starts the run, whether an approval is needed and who may give it.
The platform adds the approval step itself — you do not have to draw it into every Skill.

This page describes a preview; details can change until 3.0 is released.

## Risk levels

Every command has a risk level. The adapter declares it for each capability it publishes; where an
adapter declares none, the platform's own table applies, and a command neither rates counts as
**CRITICAL**.

| Level | What it means | Examples |
| --- | --- | --- |
| OBSERVE | Looks, changes nothing | take a photo, start the live stream, start detection |
| ADJUST | Changes a setting, nothing moves | point the gimbal, zoom, switch a light, start charging |
| MOVE | Moves the asset or a part of it | take off, fly to, return home, open or close the dock cover |
| CRITICAL | Hard or impossible to undo, or affects others | land in place, reboot, switch jamming on, point a jammer |

## The matrix

Rows are the risk levels, columns are who starts the run:

| | Operator | AI agent | Automation |
| --- | --- | --- | --- |
| **OBSERVE** | never | never | never |
| **ADJUST** | rule | rule | rule |
| **MOVE** | rule | always (at least one person) | rule |
| **CRITICAL** | rule | always (at least one person) | rule |

- **Operator** — a person in the console, at Remote Control, or through the API.
- **AI agent** — the assistant, acting for the person who uses it.
- **Automation** — a schedule, an event trigger, or the organization's automation account.

Each cell is a rule:

| Setting | Meaning |
| --- | --- |
| Approval required | No approval, or an approval before the command goes out |
| Approvers | How many different people must approve (1, 2, …) |
| Roles | An approver must hold one of these roles (built-in or your own; for the run's site or the whole organization). Empty: anyone who may approve runs (`executions:approve`) |
| Requester may approve | Off: the person who started the run — for the agent, the person it acts for — cannot approve it (four eyes) |
| Timeout | After this time without enough approvals the request expires: the command does not run, and a TIMEOUT branch drawn in a Skill takes over |

With nothing configured, operators and automation are not asked; the AI agent always waits for a
person on MOVE and CRITICAL.

## System default, organization and site

The **system admin** sets the defaults for the installation. An **organization** can set its own
rules, and a **site** inside it its own again. The most specific scope that sets a value wins —
site, then organization, then the system default — also when it is looser. Every value is inherited
on its own: an organization can change only the number of approvers of MOVE and keep everything
else from the system default.

The console shows for every value where it comes from (*built-in*, *system default*, *organization*,
*site*) and which values are locked by a hard guard.

## Hard guards

These rules are fixed in the platform and no scope can change them:

| Guard | What it does |
| --- | --- |
| Observing never waits | OBSERVE commands never need an approval |
| The AI agent never moves hardware on its own | A MOVE or CRITICAL command the agent asks for always waits for at least one person |
| The AI agent never needs less than an operator | Where an operator needs an approval, the agent needs one too, with at least as many approvers |
| Jamming needs a second person | Switching a jammer on always needs an approval by someone other than the person who asked; switching it off never waits |
| Unknown commands count as critical | A command neither its adapter nor the platform rates is CRITICAL |
| Only signed-in people approve | Services, adapters, the Integration Hub and the AI agent never approve |
| No answer means no | An unanswered request expires (after at most 24 hours) and the command does not run |
| The low-battery return never waits | The platform's own safety return home starts at once |
| No-fly zones always apply | An approval never lets a flight into a no-fly zone |

The guards look at the **higher** of a command's own level and any re-rating (see below): an
organization that re-rates `flight.takeoff` to OBSERVE still has every takeoff the agent asks for
approved by a person.

## Thresholds that raise the level

A small, fixed set of thresholds raises a command to a stricter level (MOVE or CRITICAL). Each scope
can switch them on, change them or switch them off; a scope that sets a threshold replaces the
wider scope's setting of it.

| Threshold | Raises a command when … |
| --- | --- |
| Near a no-fly zone | the asset or the command's target is inside a no-fly zone or closer to one than the given distance (if the zones cannot be read, the command counts as inside one) |
| At night | the local time (in the given time zone) is between *from* and *to* |
| Above an altitude | the command's altitude, or the asset's current height above its takeoff point, is above the given height |
| Asset type | the asset is of one of the given types (for example JAMMER) |
| Assets at once | the run commands more than the given number of assets |

OBSERVE commands are never raised.

## Re-rating a command

An organization or a site can give a command a different level than its adapter declares — for
example rate `gimbal.look_at` CRITICAL at a site with sensitive neighbours. Raising a level is a
normal change. Lowering one is a **loosening** (see below).

## Loosening the rules

A change is a **loosening** when it needs fewer approvals than the system default: a cell without an
approval where the default asks for one, fewer approvers, letting the requester approve, dropping
the required roles, re-rating a command lower, or switching a threshold off.

- Loosening needs the permission **`approval-policy:loosen`**. Organization admins do not have it
  by default; a system admin can grant it through a role.
- A **reason** is required. It is stored with the rules and shown next to them.
- The console warns before a loosening is saved and marks loosened cells.
- Optionally, the installation can require **four eyes on the change itself** (technical config
  `approval.rules.four-eyes = true`, system admin): a loosening is then stored as a proposal and
  only applies when a second person with `approval-policy:loosen` approves it in the console.

**The operator of the installation is responsible for rules loosened below the system default.**
The platform keeps the hard guards and records every loosening, who made it, when and why.

## Where the approval goes

Every way of sending a command — the console, Remote Control, the API, the AI agent, Skills and
Applications, schedules and triggers — runs through the same check. For each command of a run the
platform takes the effective cell of its level and column:

- If an approval the Skill's author drew already stands in front of the command, it is raised to
  what the rule asks (more approvers, the rule's roles, no self-approval, the shorter timeout).
- If the command can be reached without passing an approval, one approval is added in front of the
  run, asking what the strictest of its commands needs.

Only the approved way out of an approval counts: what a REJECTED or TIMEOUT branch leads to was never
approved and is checked on its own.

## Deciding an approval

The waiting run shows in the console's approval banner and on the run's page. With several approvers,
each person approves once; the run continues when enough have approved. One rejection ends the
request. The run's page shows the approvals given so far.

## Audit

Every change of the rules (who, when, the rules before and after, the reason), every proposal and
its approval, every request for an approval and every decision — including refused attempts, for
example a requester trying to approve their own run — is in the platform audit log.

## For the API

| Call | Purpose |
| --- | --- |
| `GET /api/admin-console/approval-rules?organizationId=&siteId=` | The rules of a scope with the matrix in force and the source of every value |
| `PUT /api/admin-console/approval-rules` | Store a scope's rules (`organizationId`, `siteId`, `rules`); answers `APPLIED` or `PENDING` |
| `POST /api/admin-console/approval-rules/pending/approve` | Apply a waiting loosening as the second person |
| `DELETE /api/admin-console/approval-rules/pending` | Withdraw or turn down a waiting loosening |
| `DELETE /api/admin-console/approval-rules` | Remove a scope's own rules (inherit everything) |
| `GET /api/admin-console/approval-rules/assets/{sn}` | What each command of an asset needs, per column |

The rules are stored as the technical config entry `approval.rules` (scopes GLOBAL, ORGANIZATION,
SITE). They can only be changed through the calls above; a raw technical config write of them is
refused.

## Upgrading from 2.x

The switch `risk.human-approval.required` is gone. Where it was on, the scope now has approval rules
asking for an approval on MOVE and CRITICAL commands — note that this also applies to commands an
operator sends directly, which the old switch let through. The fixed approval step the AI agent's
runs had is replaced by the agent column of the matrix.
