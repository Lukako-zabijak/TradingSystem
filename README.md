# trading system

a server-authoritative roblox trading system for exchanging items between two players. the server owns trade state, validates every request, locks inventory records during transfers, and recovers unfinished commits after a server failure.

## setup

copy the `scripts` folder into `ServerScriptService/trading_system` without changing its folders or filenames. create files marked `(server)` as server scripts and files marked `(module)` as module scripts.

before publishing:

1. review datastore names, limits, cooldowns, account age, and proximity settings in `trading_config (module)`.
2. use separate datastore names while testing in studio.
3. keep inventory changes behind `trading_server_api (module)` so receipts and transaction locks stay consistent.
4. build the client around server responses. never update client inventory optimistically.

## client requests

send one table to `ReplicatedStorage/trading/request`:

```lua
{
    version = 1,
    session = "session_from_bootstrap",
    sequence = 1,
    request_id = "request_00000000",
    action = "invite",
    payload = { target_user_id = 123456 },
}
```

available actions are `bootstrap`, `invite`, `respond`, `set_offer`, `confirm`, `cancel`, and `sync`. the server rejects extra arguments, unknown fields, malformed ids, stale sequences, oversized offers, and requests that fail current ownership or trade checks.

responses and trade updates are sent through `ReplicatedStorage/trading/push`.

## server api

trusted server scripts can use `ServerStorage/trading_server_api/invoke` for inventory reads, grants, revokes, trading access, transaction inspection, recovery, metrics, and the built-in self-tests.

reuse the same mutation id only when retrying the same grant or revoke. mutation ids make retries idempotent; they are not general request ids.

## transaction safety

the transfer path uses a recoverable two-phase commit because roblox datastores cannot atomically update two player records. both inventories are prepared first, then a durable commit or abort decision is written before inventory effects are applied. repeated recovery is safe, and a transaction that has entered `committing` is never rolled back.

the client never decides ownership, item values, capacity, or transfer results. datastore outages cause the system to fail closed. live testing is still required for datastore throttling, shutdown, disconnect, and recovery behavior in the target experience.
