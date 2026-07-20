# TradingSystem

A server-authoritative Roblox trading system split into cohesive, reviewable modules. It preserves the supplied system's versioned remote envelope, item-only offers, server-side ownership checks, session leases, revision checks, idempotent durable transactions, recovery, audit records, rate limiting, and server-only inventory API.

## Script layout

Copy the `scripts` tree into `ServerScriptService/trading_system` with the same folders and names:

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
    |-- trading_observability (module)
    |-- trading_remote_router (module)
    |-- trading_server_api (module)
    |-- trading_service (module)
    |-- trading_verification (module)
    |-- transaction_coordinator (module)
    `-- transaction_repository (module)
```

The repository files intentionally have no `.server`, `.lua`, or `.luau` suffix. Their Roblox script type is written in parentheses, as requested.

## Installation

1. Create the folder structure above in `ServerScriptService`.
2. Create each object with the type shown in its name, then paste in the matching file.
3. Enable **Studio Access to API Services** only in a safe test place when testing DataStores.
4. Do not write the system's inventory DataStore from another script. Use the bindable API below.
5. Grant globally unique item IDs. A generated ID is used when `grant_item` receives no ID.

At startup the server creates:

- `ReplicatedStorage/trading/request` — client-to-server `RemoteEvent`.
- `ReplicatedStorage/trading/push` — server-to-client `RemoteEvent`.
- `ServerStorage/trading_server_api/invoke` — server-only `BindableFunction`.
- `ServerStorage/trading_server_api/changed` — server-only `BindableEvent`.

## Client request contract

Every request uses this exact envelope; extra fields are rejected:

```lua
{
    version = 1,
    session = "server-issued-session",
    sequence = 1,
    request_id = "request_00000000",
    action = "invite",
    payload = { target_user_id = 123456 },
}
```

Supported actions and payloads:

- `bootstrap`: `{}` with session `bootstrap_session` and sequence `0`.
- `invite`: `{ target_user_id = number }`.
- `respond`: `{ trade_id = string, accept = boolean }`.
- `set_offer`: `{ trade_id = string, expected_revision = number, item_ids = { string } }`.
- `confirm`: `{ trade_id = string, expected_revision = number }`.
- `cancel`: `{ trade_id = string }`.
- `sync`: `{}`.

Clients submit intent only. They never submit item values, ownership, prices, or final transfer results.

## Server inventory API

Call `ServerStorage.trading_server_api.invoke:Invoke(action, ...)` from server code:

- `get_inventory`, player or user ID
- `grant_item`, player or user ID, item
- `revoke_item`, player or user ID, item ID
- `set_trading_enabled`, player or user ID, boolean
- `cancel_trade`, player or user ID, reason
- `inspect_transaction`, transaction ID
- `inspect_inventory`, player or user ID
- `get_durable_alerts`
- `get_metrics`
- `sweep_transactions`
- `reconcile_inventory`, player or user ID
- `run_self_tests`
- `get_self_test_report`
- `run_lease_load_test`, players, update budget, duration seconds
- `migration_status`

## Item schema

```lua
{
    id = "globally_unique_id",
    kind = "server_defined_kind",
    name = "display name",
    tradable = true,
    metadata = {},
    created_at = os.time(),
}
```

Metadata is server-private, JSON-compatible, depth-limited, node-limited, and size-limited. Clients receive only the public item fields.

## Reliability model

The commit path is a durable two-phase protocol:

1. Persist a transaction record.
2. Mark both inventories as prepared under their active server leases.
3. Move the transaction to `committing`.
4. Apply each side idempotently and record a receipt.
5. Mark the transaction `completed`.

Before `committing`, a failed prepare is rolled back. After `committing`, recovery only finishes missing applications. The open-transaction index, MessagingService notification, join recovery, and periodic sweep all converge on the same idempotent recovery path.

## Verification

Pure startup checks cover successful commit, prepare rollback, partial-commit recovery, strict payload rejection, and offer normalization. Startup stops before remotes are connected if a self-test fails. For full validation, test in a separate Roblox place with API services enabled and exercise two real clients, forced disconnects, and server shutdown during commit.
