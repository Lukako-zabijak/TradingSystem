# TradingSystem

TradingSystem is a server-authoritative Roblox trading package for item transfers between two players. It provides strict remote validation, inventory leases, revisioned offers, a recoverable two-phase commit protocol, durable mutation journals, bounded abuse controls, durable transaction records, and a server-only inventory API.

This repository is the authoritative server package and protocol documentation. It does not include a client ui/controller, and it does not claim that any distributed system is 100% failure-proof. Read [TESTING.md](TESTING.md) and validate the package in a separate published test place before release.

## Script layout

Copy the complete `scripts` tree into `ServerScriptService/trading_system` with the same folders and names:

```text
trading_system
|-- trading_server (server)
|-- shared
|   |-- trading_config (module)
|   |-- trading_schema (module)
|   |-- trading_types (module)
|   `-- trading_utility (module)
`-- server
    |-- data_store_gateway (module)
    |-- inventory_repository (module)
    |-- trade_protocol (module)
    |-- trade_transfer_rules (module)
    |-- trading_observability (module)
    |-- trading_remote_router (module)
    |-- trading_server_api (module)
    |-- trading_service (module)
    |-- trading_verification (module)
    |-- transaction_coordinator (module)
    `-- transaction_repository (module)
```

The repository files intentionally have no `.server`, `.lua`, or `.luau` suffix. The Roblox script type appears in parentheses in each filename.

## Installation

1. Create `ServerScriptService/trading_system` and reproduce the complete tree above. `trade_transfer_rules (module)` is required by `inventory_repository (module)` and must not be omitted.
2. Create `trading_server (server)` as a server script and every `(module)` file as a module script.
3. Use a separate published test place and separate datastore names while validating migrations and fault behavior. Enable Studio access to api services only for that test place.
4. Review `trading_config (module)` before release. In particular, choose store/topic names, inventory and offer limits, account-age policy, proximity policy, recovery concurrency, and remote-rate limits for the experience.
5. Do not modify the inventory, mutation, transaction, or open-index datastores from another script. Use the server API so leases, revisions, journals, receipts, reservations, and pending operations remain coherent.
6. Integrate a client that follows the request contract and treats every server response as authoritative. The session value is for connection freshness and replay isolation; it is not a secret that makes a compromised client trustworthy.
7. Back up and exercise schema migration against representative version 1 and version 2 records before rolling inventory and transaction schema version 3 into an existing experience. Shut down all legacy writer servers before allowing the migration to complete; a mixed-version rollout can create an unindexed legacy transaction after the one-time full scan. Startup runs a fenced, retryable full transaction-key reindex and keeps dual-reading the legacy `open_transactions` index.

At startup the server creates:

- `ReplicatedStorage/trading/request` - the client-to-server remote event;
- `ReplicatedStorage/trading/push` - the server-to-client remote event;
- `ServerStorage/trading_server_api/invoke` - the server-only bindable function;
- `ServerStorage/trading_server_api/changed` - the server-only bindable event.

Only trusted server scripts should be able to call the bindable API. Code running on the server is inside the economy trust boundary.

## Client request contract

Each client call must contain exactly one table argument. Unknown envelope fields, unknown payload fields, and extra remote arguments are rejected.

```lua
{
    version = 1,
    session = "server_issued_session",
    sequence = 1,
    request_id = "request_00000000",
    action = "invite",
    payload = { target_user_id = 123456 },
}
```

Supported actions and payloads:

- `bootstrap`: `{}` with session `bootstrap_session` and sequence `0`;
- `invite`: `{ target_user_id = number }`;
- `respond`: `{ trade_id = string, accept = boolean }`;
- `set_offer`: `{ trade_id = string, expected_revision = number, item_ids = { string } }`;
- `confirm`: `{ trade_id = string, expected_revision = number }`;
- `cancel`: `{ trade_id = string }`;
- `sync`: `{}`.

After bootstrap, sequence numbers must advance by exactly one. Reusing a cached request ID is accepted only when the sequence and the entire normalized request match the original. The retry cache is deliberately bounded and session-local; it is not a durable business idempotency store.

Clients submit intent only. They never submit item values, authoritative ownership, capacity, or transfer results.

## Abuse controls

The remote boundary currently applies:

- exact argument-count and exact table-key validation;
- bounded ASCII identifiers, finite safe integers, dense offer arrays, duplicate rejection, and offer-size limits;
- server-derived player identity, server-issued session matching, monotonic sequence checks, and request replay comparison;
- a server-wide token bucket plus per-session and per-action token buckets;
- security scoring with decay and a configured kick threshold;
- target invitation cooldowns and generic target-unavailable responses;
- account-age checks and optional server-side proximity checks;
- synchronous invite/trade expiry checks rather than relying only on delayed callbacks;
- per-user reservations around trusted inventory mutations;
- sampled and deduplicated security alerts with bounded telemetry workers;
- bounded commit recovery workers and bounded recoveries per sweep.

These controls reduce accidental and malicious load, but they do not replace platform-level monitoring, game-specific eligibility rules, moderation, or server-authoritative movement validation.

## Server inventory API

The server bindable accepts these actions:

- `get_inventory`, player or user ID;
- `grant_item`, player or user ID, item, optional mutation ID;
- `revoke_item`, player or user ID, item ID, optional mutation ID;
- `set_trading_enabled`, player or user ID, boolean, optional mutation ID;
- `cancel_trade`, player or user ID, reason;
- `inspect_transaction`, transaction ID;
- `inspect_inventory`, player or user ID;
- `get_durable_alerts`;
- `get_metrics`;
- `sweep_transactions`;
- `reconcile_inventory`, player or user ID;
- `run_self_tests`;
- `get_self_test_report`;
- `run_lease_load_test`, players, update budget, duration seconds;
- `migration_status`.

An optional mutation ID should be generated once by the trusted calling service and reused only when retrying the exact same logical grant, revoke, or settings change. Every call receives a mutation ID; when none is supplied, the service creates one. The system writes a durable operation record before effects, writes the effect with an inventory receipt and pending-mutation marker, marks the operation completed, and clears the marker last. Reuse with a different user, kind, payload, or result is a hard conflict. The operation journal is not pruned by this package, so delayed exact retries do not depend on the bounded inventory receipt cache.

## Item contract and IDs

With the default `require_server_generated_item_ids = true`, callers must omit `id` when granting an item:

```lua
{
    kind = "server_defined_kind",
    name = "display name",
    tradable = true,
    metadata = {},
    created_at = os.time(),
}
```

The service generates the item ID. When a valid mutation ID is supplied, the generated item ID is deterministic for that mutation; otherwise it is generated from a new guid. Caller-supplied item IDs are rejected under the default policy. If `created_at` is omitted, the first journaled value becomes canonical and is reused by exact retries.

Metadata remains server-private. It must be JSON-compatible and is bounded by depth, node count, string length, and encoded size. Clients receive only public item fields.

## Reliability model: two-phase commit is retained

The hardened design deliberately retains a durable two-phase commit protocol because Roblox datastores do not provide one atomic write spanning two player inventory keys.

```text
preparing -> aborting -> aborted
preparing -> committing -> completed
```

The protocol is monotonic:

1. Reserve a slot in the sharded open-transaction index, then persist a `preparing` transaction with the two immutable item snapshots and a coordinator fencing lease.
2. Prepare both inventories under their active inventory leases. Preparation revalidates ownership, tradability, item identity, incoming ID collisions, and projected capacity against the authoritative record, then stores the pending transaction ID.
3. Persist the decision before its effects:
   - persist `aborting` before clearing prepared reservations;
   - persist `committing` before removing or adding any item.
4. In `committing`, apply each participant postimage idempotently. The transfer rules run again inside the datastore update, and each inventory stores a transaction receipt before the apply is considered successful.
5. Persist `completed` only after both receipts exist.
6. Finalize both inventories after completion by clearing the pending transaction only when that inventory has its receipt. Terminal finalization is idempotent and is also performed by recovery.
7. Remove the transaction from the open index only after terminal cleanup succeeds. The abort path similarly reaches `aborted` before unindexing.

An uncertain datastore response is not interpreted as a decision. The protocol performs a cache-disabled consistent read and follows the durable state it observes. It never rolls back a transaction observed in `committing`, and it never applies transfer effects before a durable commit decision.

Coordinator ownership is fenced by owner, token, epoch, and expiry. Recovery can claim an expired coordinator lease; every renewal, state transition, and progress write must match the current fence, preventing a stale worker from continuing after takeover.

The open index is sharded and non-evicting for valid unresolved transactions. A full shard rejects a new transaction instead of silently dropping an older recovery obligation. The default configuration has 16 shards with 512 entries per shard, but distribution is hash-based and each shard is independently bounded. Existing deployments are protected by a permanent legacy-index dual read plus a one-time lease-fenced, paginated scan of every `tx_` key; a legacy marker is removed only after its shard marker is confirmed.

## Operational limitations

- No implementation can guarantee 100% availability or bounded recovery while datastores remain unavailable. The protocol is designed to fail closed and converge when required services recover.
- Two-phase commit provides durable decisions and eventual convergence, not a single atomic multi-key datastore operation. A participant inventory remains unavailable for conflicting mutations while its pending transaction is unresolved.
- The open index is finite. When a shard is full, new transactions mapped to that shard fail until entries are reconciled.
- Upgrading from the legacy bounded index requires a hard old-server cutover before the one-time full scan completes. Do not run old and new persistence writers concurrently during that migration.
- Transaction and inventory mutation receipts are bounded, but trusted inventory operations also use a separate durable operation journal for long-horizon idempotency. That journal is still not a substitute for a platform purchase receipt source of truth or a game-specific item-origin ledger.
- The request replay cache is in memory, bounded, and reset by a new player session or server process.
- `require_proximity` defaults to false. Experiences that require local trades should enable it and pair it with their own movement and world-state validation.
- Rate limits and security scores are process/session local. Cross-server abuse detection requires separate operational infrastructure.
- Durable alerts and audits are best effort and worker-bounded; telemetry may be dropped under overload rather than consuming unbounded server resources.
- The server bindable API trusts server code. Vet third-party server scripts and packages before placing them in the experience.
- The package does not provide client ui, moderation policy, cross-server trading, marketplace receipt processing, or a permanent item-origin ledger.

## Verification

Startup checks are a narrow deterministic gate, not proof of live datastore correctness. Follow the fault-injection matrix in [TESTING.md](TESTING.md), record evidence for the target place and configuration, and repeat it after schema, datastore, inventory, or transaction changes.
