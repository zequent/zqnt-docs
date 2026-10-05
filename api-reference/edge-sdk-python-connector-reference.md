# Edge SDK (Python) — Connector API Reference

> For the conceptual introduction to this model, see
> [Applications & Skills](../concepts/applications-and-skills.md). For 1.3.x (end of life), see the
> [1.3 Connector reference](edge-sdk-python-connector-reference-1.3.md).

Exhaustive method reference for `ConnectorClient`.

## What changed from 1.3

- **`get_mission`/`get_task`/`get_task_by_flight_id` are gone — not deprecated, just removed.**
  There's no backend RPC left for any of them; the methods don't exist on `ConnectorClient` at all
  (this SDK doesn't keep the stub-that-raises pattern the client SDK's Python
  `MissionAutonomyClient` uses for its own retired Mission/Task methods — it just deletes them
  outright, matching the Java Edge SDK's clean-removal approach). The 1.3 reference already found
  [no confirmed real-adapter usage](edge-sdk-python-connector-reference-1.3.md#missions-and-tasks)
  of any of the three, so this is unlikely to affect a real adapter migrating forward.
- **`register_asset` is gone, replaced by pairing with a claim code** — `ensure_asset` and
  `redeem_asset_claim`.
- **Calls carry the adapter's edge credential.** The constructor takes `token`, which defaults to
  `ZQNT_EDGE_TOKEN`; the platform refuses calls without one.
- **New: Skill Registry self-reporting** — four methods, working with the raw generated
  `SkillContractProtoDTO` rather than a plain-Python model (the contract shape is already large and
  typed; wrapping it a second time buys little for what's normally a write-once-per-command call).
- `ConnectorClient` has no Scheduler methods, as on 1.3 — that's `MissionAutonomyClient`'s job,
  and it *is* affected by a cross-cutting `SchedulerDTO` reshape; see
  [Mission Autonomy — Scheduler lookup](../edge-sdk/edge-sdk-python-mission-autonomy.md#scheduler-lookup).

## Constructor

`ConnectorClient(host, port=50053, call_timeout=30.0, max_retries=3, claim_code=None, token=None)`

- `claim_code` — the one-time claim code `ensure_asset` redeems when the asset is unknown.
  `EdgeAdapterRuntime` sets it from `ZQNT_CLAIM_CODE`.
- `token` — the adapter's edge credential; `None` reads `ZQNT_EDGE_TOKEN`.
- The default `port` is not the platform's real Connector Service port (`8010`) — always set it.

## Lifecycle

| Method | Returns | Purpose |
| --- | --- | --- |
| `connect()` | `None` | Open the gRPC channel and initialize the stub |
| `close()` | `None` | Close the channel and release resources |

## Assets

| Method | Returns | Purpose |
| --- | --- | --- |
| `get_asset_by_sn(sn)` | `Asset \| None` | Look up an asset by serial number; `None` if not found |
| `ensure_asset(asset)` | `Asset \| None` | Make sure the asset's serial number exists on the platform: return it if it does, otherwise redeem the configured `claim_code` for it. With no claim code it reports what it found and creates nothing |
| `redeem_asset_claim(code, asset)` | `Asset \| None` | Trade a one-time claim code for an asset, and return it; `None` when the code is refused |
| `watch_assets()` | `AsyncIterator[list[Asset]]` | Subscribe to the platform's asset-monitoring stream. **The 2.0.0 platform does not implement this stream** and answers it with an error |

**Pairing.** Call `ensure_asset` at startup. Because a claim is single-use, the lookup comes first:
after the first successful pairing there is nothing left to redeem, and the adapter would otherwise
log a refusal on every restart. Without a claim code it creates nothing — the normal state for an
adapter whose assets are provisioned in the Admin Console.

`redeem_asset_claim` is the one call an adapter makes without any platform identity: the code *is*
the credential. The organization that owns the created asset comes from the claim, not from
`asset`, whose `organization` field is ignored server-side. Every refusal looks the same —
unknown, expired, revoked, exhausted, or not valid for this kind of device — so the call cannot be
used to discover which codes exist.

## Skill Registry — new in 2.0.x

| Method | Returns | Purpose |
| --- | --- | --- |
| `observe_skill_contract(contract)` | `SkillContractProtoDTO \| None` | Upsert `contract` — new for a never-seen `(command_id, schema_version)` pair, or refreshes content/last-seen for one already known |
| `list_skill_contracts(status=None, command_id=None)` | `list[SkillContractProtoDTO]` | List the registry, optionally filtered by `status` (a `SkillContractStatus` enum value). When `command_id` is set, returns that one command's full version history instead — `status` is then ignored |
| `set_skill_contract_status(contract_id, status)` | `SkillContractProtoDTO \| None` | Move a contract through its lifecycle: `ACTIVE` / `DRAFT` / `DEPRECATED` / `RETIRED` |
| `set_skill_contract_permissions(contract_id, required_permissions)` | `SkillContractProtoDTO \| None` | Full replacement, not a merge. **Declarative only — nothing enforces these permissions yet** |

## Error handling

The asset and Skill Registry methods return `None` (or `[]` for `list_skill_contracts`) on a
business-level failure (`resp.has_errors` — not found, refused, validation) and log it, rather than
raising — unlike the client SDK's `Application`/`SkillExecution` methods, which raise on failure.
Transport-level failures (`UNAVAILABLE`, timeouts, ...) raise `grpc.aio.AioRpcError`.

## Retry and timeout behavior

Every call carries a per-call deadline (`call_timeout`, default 30.0s) and is retried automatically
on `UNAVAILABLE`/`DEADLINE_EXCEEDED` — every other gRPC error propagates immediately, unretried:

- Up to `max_retries` attempts (default 3, set in the constructor).
- Exponential backoff starting at 0.5s, doubling each retry, capped at 10s.
- Each call logs its transaction id (`tid`, a fresh UUID per call) so failures can be correlated
  across services.

## See also

- [Applications & Skills](../concepts/applications-and-skills.md) — narrative introduction
- [Java reference](edge-sdk-connector-reference.md)
- [1.3 Connector reference](edge-sdk-python-connector-reference-1.3.md) — the end-of-life 1.3
  interface
