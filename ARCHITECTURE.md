# Architecture

## Authority boundary

The server owns inventory records, eligibility, live trade state, immutable transfer snapshots, revisions, leases, transaction decisions, and final transfer results. A client can submit only a versioned intent packet. The player argument supplied by the remote event is the identity boundary; a client-provided session value is used for stale-session and replay isolation, not as proof that client code is trusted.

The router rejects extra remote arguments, unknown table keys, malformed arrays, invalid or oversized identifiers, non-finite numbers, session mismatches, request reuse with different content, replayed or out-of-order sequences, and rate-limit violations before dispatching domain actions.

## Module responsibilities

| Script | Responsibility |
| --- | --- |
| `trading_server (server)` | Creates Roblox instances, wires dependencies, runs startup verification, and owns shutdown order. |
| `trading_config (module)` | Immutable names, schema/protocol versions, limits, retry policy, lease policy, recovery concurrency, and abuse thresholds. |
| `trading_types (module)` | Shared static contracts for items, inventories, transactions, coordinator fences, and player sessions. |
| `trading_utility (module)` | ID generation, bounded identifier and number checks, deep copy/equality, exact-key checks, and receipt pruning. |
| `trading_schema (module)` | Runtime validation, normalization, migrations, persisted-record validation, public projections, and client request contracts. |
| `data_store_gateway (module)` | Budget-aware bounded retries, cache-disabled consistent reads for ambiguity resolution, and persistence metrics. |
| `trade_transfer_rules (module)` | Pure participant-side ownership, tradability, collision, and projected-capacity validation. |
| `inventory_repository (module)` | Serialized session access, inventory leases, optimistic revisions, durable mutation journaling, preparation, rollback, idempotent apply, and terminal finalization. |
| `transaction_repository (module)` | Fenced monotonic state transitions, coordinator lease claims/renewals, progress records, sharded and legacy open indexing, crash-retryable reindex migration, consistent reads, and audit hooks. |
| `trade_protocol (module)` | Pure two-phase decision, ambiguity reconciliation, abort, commit, recovery, and finalization rules. |
| `transaction_coordinator (module)` | Creates transaction snapshots, binds protocol operations to repositories, fences workers, limits recovery concurrency, and sweeps open transactions. |
| `trading_service (module)` | Player lifecycle, in-memory invitations/trades, offers, confirmations, expiry, per-user reservations, commit watching, and trusted inventory operations. |
| `trading_remote_router (module)` | Exact remote contract, server/session/action token buckets, replay cache, security score, and request dispatch. |
| `trading_server_api (module)` | Narrow server-only bindable API, including optional mutation IDs for retryable trusted writes. |
| `trading_observability (module)` | Sampled alerts, bounded audit workers, metrics, durable best-effort telemetry, and operational events. |
| `trading_verification (module)` | Deterministic protocol and trust-boundary regression checks. |

## Dependency direction

```text
trading_server
  -> trading_service / trading_remote_router / trading_server_api
      -> transaction_coordinator
          -> trade_protocol
          -> inventory_repository
              -> trade_transfer_rules
          -> transaction_repository
              -> data_store_gateway
      -> trading_schema / trading_config / trading_utility
```

The protocol is pure and receives persistence operations from the coordinator. Repositories own platform persistence. Transfer rules are pure and are called before preparation and again inside authoritative apply updates. The service owns player and live-trade lifecycle. The router is the only client request boundary.

## Two-phase transaction state machine

Two-phase commit is retained by design.

```text
preparing -> aborting -> aborted
preparing -> committing -> completed
```

The only valid forward transitions are:

- `preparing -> aborting -> aborted`;
- `preparing -> committing -> completed`.

There is no transition from `committing` to an abort state. There is no transition out of a terminal state.

### Durable decision before effects

Preparation places a reversible pending-transaction reservation on each inventory. After preparation, the transaction record stores the decision before the decision's effects occur:

- rollback starts only after `aborting` is durable;
- item removal/addition starts only after `committing` is durable.

If a decision write returns an error, the protocol uses a cache-disabled consistent read. It follows the durable state rather than guessing from the error. An observed `committing` transaction is always driven forward. An observed `preparing` or `aborting` transaction is driven through the abort path.

### Coordinator fencing lease

Every nonterminal transaction contains:

- a coordinator owner;
- an unguessable coordinator token;
- a monotonically increasing epoch;
- an expiry timestamp.

The active coordinator renews the lease during protocol progress. After expiry, a recovery worker can claim the transaction and increment the epoch. Renewals, transitions, and progress writes compare the complete fence. A former owner with a stale token or epoch cannot mutate transaction state after takeover.

The coordinator also bounds active recovery workers and recoveries started by a sweep. A protected worker cleanup path clears worker accounting even when protocol code raises an error.

### Idempotent apply and terminal finalization

For each participant, apply performs these checks against the current authoritative inventory record:

- the record belongs to the participant side;
- every outgoing item still exists, is tradable, and equals the snapshotted item;
- no incoming item ID collides with an existing item;
- projected count remains between zero and `maximum_inventory_items`;
- the expected pending transaction is present;
- an active inventory lease is respected.

The same transfer rules are applied during prepare and inside the apply update. A successful apply stores a transaction receipt in the same inventory update as the item changes. A retry that observes the receipt is successful without applying the transfer twice.

Apply deliberately leaves the pending transaction in place. After both sides are applied, the coordinator persists `completed`. Terminal finalization then clears each pending marker only if the matching receipt exists. If the server stops after completion but before cleanup, recovery re-enters `completed`, finalizes both sides idempotently, and only then removes the open-index entry.

The abort path similarly persists `aborting`, clears pending reservations idempotently, persists `aborted`, and unindexes the terminal transaction.

## Open-transaction index

The recovery index is sharded by a deterministic hash of the transaction ID. Valid unresolved entries are never evicted to make room. Each shard has a hard capacity and rejects a new entry when full. This provides backpressure instead of losing a recovery obligation.

Indexing occurs before transaction creation. If creation never becomes durable, consistent recovery reads identify the missing transaction and remove the harmless index entry. Nonterminal renewals, transitions, and progress writes touch the index timestamp. Sweeps read all shards consistently and start only bounded, sufficiently old recoveries.

Upgrades permanently dual-read the legacy `open_transactions` key. A lease-fenced startup migration first drains legacy markers into confirmed shards, then paginates every `tx_` datastore key and reindexes each normalized nonterminal transaction. It aborts without marking completion after any ambiguous page advance, read, normalization, or index failure; another pass restarts safely. Legacy entries are deleted only after their shard marker is confirmed. Because the full scan is deliberately one-time, deployment must retire every legacy writer before migration completes.

## Inventory and mutation invariants

- A player has at most one active inventory lease across servers.
- All session-backed repository operations serialize through the session lock.
- A user participates in at most one live trade per server.
- In-memory reservations close the check-then-yield window around trusted grants, revocations, and settings writes.
- A pending invitation does not block a trusted inventory mutation; an active or committing trade does.
- Offer changes increment the trade revision and clear both confirmations.
- Both parties must confirm the same revision.
- Empty two-sided offers do not commit.
- Item IDs are unique across both transaction snapshots.
- Capacity and collision rules are checked against the datastore preimage, not trusted from a client or an earlier in-memory snapshot.
- A prepared inventory names at most one pending transaction.
- A regular mutation carries a globally unique operation ID. The durable order is operation intent, atomic inventory effect plus receipt and pending marker, operation completion, then pending-marker finalization.
- Mutation ID reuse with a different user, kind, payload, or result fails closed. Exact retries apply the first persisted canonical payload, including server-generated item fields.
- Pending transactions and pending mutations are mutually exclusive. Player load recovers a pending mutation before transaction recovery and before the session becomes ready.
- With the default policy, grant callers cannot choose item IDs; the service generates them.

## Remote and abuse model

The router combines a server-wide token bucket with per-session and per-action buckets. It requires one request argument, validates the complete envelope and action payload, and consumes monotonic sequence numbers only after validation and session matching. A cached request ID may be replayed only with the identical normalized request and sequence.

The service adds generic target-unavailable responses, a target invitation cooldown, account-age policy, optional proximity policy, explicit deadline checks, participant-state revalidation, and per-user reservations. These controls limit one-server abuse; they do not provide cross-server reputation or replace game-specific anti-cheat and moderation.

Observability samples repeated remote violations, prunes its dedupe table, caps alert and audit workers, and counts dropped telemetry. Under overload, diagnostic telemetry is allowed to drop instead of creating unbounded persistence work. Transaction records and inventory receipts, not telemetry, are the correctness source of truth.

## Lifecycle ownership

`trading_service (module)` owns player connections, messaging subscriptions, lease refresh, index migration and sweep loops, session bindable events, reservations, commit watchers, and in-memory trade maps. Player load recovers pending mutation state first and transaction state second before becoming ready. Player removal marks a session inactive, closes noncommitting trades, releases its inventory lease through serialized repository access, destroys session resources, and starts recovery when a pending transaction remains.

`transaction_coordinator (module)` owns active-transaction markers and bounded recovery workers. `stop` prevents new recovery work. Shutdown disconnects request handling, stops server API invocation, stops service loops, releases leases, and gives pending recovery a bounded best-effort window.

## Failure semantics and trust assumptions

- Datastore failures are expected and can have ambiguous outcomes; consistent reads and receipts reconcile them.
- Recovery is convergent but cannot promise a completion deadline during a continuing platform outage.
- Multi-key visibility is not instantaneous. Conflicting inventory operations remain closed while a pending transaction exists.
- A full open-index shard rejects new commits; it does not evict unresolved work.
- Receipt tables are bounded. The separate mutation operation journal is durable and unpruned by this package, but it is not a platform purchase receipt or a game-specific item-origin ledger.
- The server bindable API assumes calling server scripts are trusted.
- Telemetry and messaging notifications are best effort. Datastore transaction and inventory records remain authoritative.
- Proximity is optional and disabled by default; a production game must choose and test the correct eligibility policy.
