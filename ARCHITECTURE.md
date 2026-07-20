# Architecture

## Authority boundary

The server owns inventory records, eligibility, trade state, item snapshots, revisions, transaction state, leases, and transfer results. The client can only submit a versioned intent packet. The remote router rejects unknown keys, malformed arrays, invalid IDs, replays, out-of-order sequences, session mismatches, and rate-limit violations before domain code runs.

## Module responsibilities

| Script | Responsibility |
| --- | --- |
| `trading_server (server)` | Creates Roblox instances, wires dependencies, runs startup verification, and owns shutdown order. |
| `trading_config (module)` | Immutable limits, names, versions, and timing policy. |
| `trading_types (module)` | Shared static contracts for items, records, transactions, leases, and sessions. |
| `trading_utility (module)` | IDs, safe-number/string checks, deep copy/equality, exact-key checks, and bounded receipt pruning. |
| `trading_schema (module)` | Runtime validation, normalization, migrations, public projections, and client request contracts. |
| `data_store_gateway (module)` | Budget-aware bounded retries and persistence metrics. |
| `inventory_repository (module)` | Session leases, optimistic revisions, mutation receipts, transaction preparation, rollback, and idempotent transfer application. |
| `transaction_repository (module)` | Durable transaction state machine, progress, audit hooks, and open-transaction index. |
| `trade_protocol (module)` | Pure two-phase commit and recovery rules. |
| `transaction_coordinator (module)` | Connects the pure protocol to repositories, starts recovery workers, and sweeps open transactions. |
| `trading_service (module)` | Player lifecycle, in-memory trade state, offers, confirmations, completion, and server inventory operations. |
| `trading_remote_router (module)` | Authentication sequence, replay cache, token buckets, security score, and request dispatch. |
| `trading_server_api (module)` | Narrow server-only bindable API. |
| `trading_observability (module)` | Bounded durable alerts, audit events, metrics, and operational events. |
| `trading_verification (module)` | Deterministic protocol and trust-boundary regression checks. |

## Dependency direction

```text
bootstrap
  -> service / router / server api
      -> coordinator
          -> pure protocol
          -> inventory repository
          -> transaction repository
              -> data store gateway
      -> schema / config / utility
```

The pure protocol does not depend on Roblox services. Repositories own persistence. The service owns players and live trade state. The router owns the client trust boundary. This prevents remote parsing, persistence callbacks, and trade rules from becoming one circular module.

## Transaction invariants

- A user participates in at most one live trade per server.
- Both offers are revalidated against authoritative item snapshots before preparation.
- A prepared inventory names exactly one pending transaction.
- A transaction cannot move backward after entering `committing`.
- Applying a transaction twice is safe because each inventory stores its transaction receipt.
- A regular grant, revoke, or setting change is retry-safe because it stores a mutation receipt.
- An active, unexpired lease prevents another server from changing that player's inventory.
- Recovery may abort an incomplete `prepared` transaction, but it only completes a `committing` transaction.

## Lifecycle ownership

`trading_service (module)` owns player connections, MessagingService subscriptions, lease refresh tasks, the sweep task, session bindable events, and in-memory trade tables. `stop` disconnects subscriptions, stops new loop iterations, releases leases, and gives pending recoveries a bounded shutdown window.
