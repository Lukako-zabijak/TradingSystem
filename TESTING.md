# Testing and fault-injection matrix

## Evidence standard

Passing startup checks is necessary but not sufficient. They exercise pure protocol and schema behavior with in-memory fakes; they do not reproduce live datastore ambiguity, cache behavior, request budgets, multiple servers, player disconnect timing, or platform outages.

Do not describe this package as 100% correct or production-proven from static review or startup checks alone. Record the place version, configuration, server logs, transaction records, both inventory preimages/postimages, receipts, open-index entries, and test result for every release candidate.

Never run destructive fault injection against production store names or real player inventories.

## Built-in deterministic gate

`trading_verification (module)` is the startup gate exposed through `run_self_tests` and `get_self_test_report`. Its deterministic checks currently cover:

- successful execution through terminal finalization and unindexing;
- abort-decision ordering after a prepare denial;
- cache-disabled readback after an acknowledged-but-ambiguous commit decision;
- fail-closed behavior when the commit decision cannot be read;
- recovery from `preparing`, `committing`, and `completed`;
- preservation of `committing` after an apply failure;
- coordinator fence loss before participant effects;
- schema-2 partial-commit migration and idempotent roll-forward;
- valid, empty, and cross-participant-duplicate transaction schemas;
- durable mutation-operation normalization, canonical generated timestamps, and pending-operation exclusion;
- transfer capacity boundaries and incoming ID collisions;
- offer normalization plus malformed, unknown, unsafe-number, and bootstrap request rejection.

The bootstrap server does not connect remotes when this gate reports failure unless `disable_self_tests` is deliberately set. That attribute is for targeted diagnosis only and must not be enabled in a release build.

The built-in gate does not replace the matrix below. If the verification module changes, inspect the report's individual test names and failures rather than relying only on its aggregate boolean.

## Core invariants to assert in every transaction test

- Item ownership is conserved: each transferred item exists in exactly one participant inventory after completion and remains with its original owner after abort.
- No inventory exceeds `maximum_inventory_items`.
- No incoming item overwrites an existing item with the same ID.
- No item effect occurs before a durable `committing` decision.
- No rollback effect occurs before a durable `aborting` decision.
- Once `committing` is observed, recovery never chooses abort.
- Repeating apply does not repeat item changes because the receipt is checked in the same update.
- `completed` transactions are finalized on both inventories before their open-index entry is removed.
- `aborted` transactions leave no pending marker and do not contain transfer effects.
- A stale coordinator fence cannot renew, mark progress, or transition a transaction.
- Every nonterminal transaction remains discoverable in its open-index shard unless creation itself is consistently absent.
- A trusted mutation records intent before effects, records completion after a receipt-bearing effect, and clears its pending marker only after completion.
- An inventory never contains both a pending transaction and a pending mutation.

## Fault-injection matrix

`not recorded` means the scenario must be executed and evidence attached for the target release; it is not a claim that the scenario currently fails.

| Scenario | Harness and injected fault | Required result | Current evidence |
| --- | --- | --- | --- |
| clean two-party commit | deterministic fake, then two live clients | `preparing -> committing -> completed`; one receipt per inventory; both pending markers finalized; index removed | deterministic protocol path covered; live result not recorded |
| second prepare denied | fake repository rejects participant two | `preparing -> aborting -> aborted`; participant one reservation rolls back; no item moves | deterministic decision ordering covered; persisted-state result not recorded |
| empty trade | both offers empty and both clients confirm | commit is rejected before transaction creation | manual result not recorded |
| outgoing item changed | revoke or alter an offered item before prepare | prepare fails closed and aborts; no other inventory changes | fault harness required |
| incoming ID collision | recipient already owns the incoming ID | prepare/apply rejects; no overwrite and no partial success | pure rule covered; repository integration not recorded |
| capacity boundary | recipient starts at the maximum and would gain a net item | transfer is rejected inside the authoritative update; stored record remains normalizable | pure over-capacity rule covered; repository integration not recorded |
| capacity-safe exchange | full recipient sends and receives the same count | projected capacity passes; each item remains uniquely owned | pure boundary rule covered; repository integration not recorded |
| response lost after prepare write | fake gateway commits pending marker, then returns failure | consistent inventory read recognizes preparation; no duplicate reservation | fault harness required |
| response lost after `aborting` decision | fake gateway persists `aborting`, then returns failure | consistent transaction read drives rollback and reaches `aborted` | fault harness required |
| response lost after `committing` decision | fake gateway persists `committing`, then returns failure | consistent read drives commit forward; rollback is never called | deterministic protocol ambiguity covered; datastore integration not recorded |
| response lost after first apply | first inventory update stores item effects and receipt, then reports failure | consistent read finds the receipt; retry performs no second item mutation | fault harness required |
| crash after first apply | stop the server after one receipt but before the second apply | join recovery or sweep applies only the missing side and reaches `completed` | live result not recorded |
| crash after `completed`, before finalization | persist completion, then stop before clearing one or both pending markers | recovery finalizes both receipt-bearing inventories and then unindexes | pure completed recovery covered; live result not recorded |
| abort crash after first rollback | stop after one pending marker is cleared in `aborting` | recovery clears the other marker and reaches `aborted` without item effects | live result not recorded |
| coordinator takeover | worker one lease expires; worker two claims a higher epoch; worker one resumes | every stale renewal/progress/transition is rejected; only worker two advances state | multi-server fake and live result not recorded |
| active inventory lease conflict | join the same user on another server while the first lease is current | second acquire is refused and cannot mutate the inventory | two-server result not recorded |
| expired inventory lease during operation | delay persistence beyond lease expiry | operation fails closed or reconciles under a valid lease; no stale session write wins | fault harness required |
| open-index shard full | fill one shard with valid unresolved entries | the next mapped transaction is rejected; no existing valid entry is evicted | fake-store result not recorded |
| index entry without transaction | allow index reservation, then prevent transaction creation | consistent recovery observes `not_found` and removes only the ghost entry | fault harness required |
| terminal transaction still indexed | prevent finalization or unindex once | later recovery repeats finalization safely and removes the entry only after cleanup | fault harness required |
| legacy index migration | seed schema-2 transactions in `open_transactions`, including a partial `committing` transaction omitted by the old bounded index | dual-read discovers legacy markers; full `tx_` scan reindexes every nonterminal record; marker removal follows confirmed shard write | schema-2 roll-forward covered; datastore migration result not recorded |
| mixed-version cutover | keep one legacy writer alive while staging the new version | release procedure blocks migration completion until every legacy server is retired; no old writer can create post-scan work | deployment drill required |
| ambiguous migration page advance | make `AdvanceToNextPageAsync` advance and then report failure | migration does not mark complete; a later pass restarts from the first page and remains idempotent | live result not recorded |
| corrupted transaction record | inject unknown keys, malformed progress, duplicate item IDs, invalid fence, or invalid terminal time | normalization rejects before any inventory mutation; unresolved index entry remains available for operator repair | duplicate/empty schema checks covered; broader fuzz result not recorded |
| mutation retry | repeat one grant, revoke, or setting operation with the same mutation ID after more than 64 later mutations | durable completed operation returns the stored result without a second mutation, even after its inventory receipt is pruned | canonical operation schema covered; repository integration not recorded |
| generated grant fields | retry a grant that omitted `created_at` after wall-clock time advances | first persisted item payload remains canonical and no self-conflict or duplicate grant occurs | canonical timestamp comparison covered; repository integration not recorded |
| mutation ID misuse | attempt to reuse one mutation ID for a different user, kind, payload, or result | global durable operation identity rejects reuse as `mutation_id_conflict` | caller integration test required |
| response lost after mutation effect | persist inventory effect, receipt, and pending marker, then report failure | uncached read proves the effect; operation is completed and pending marker is finalized without repeating the effect | fault harness required |
| crash after mutation effect | stop after receipt-bearing effect but before operation completion | load recovery completes the durable operation, clears the pending marker, and exposes the inventory only afterward | live result not recorded |
| crash after mutation completion | stop after operation completion but before pending-marker finalization | load recovery verifies completed status and clears the marker idempotently | live result not recorded |
| caller-supplied item ID | grant with `id` while server-generated IDs are required | request is rejected; successful grants return a server-generated ID | server-api result not recorded |
| malformed remote | send unknown fields, sparse arrays, duplicate IDs, oversized strings, non-finite numbers, invalid session, and extra remote arguments | request is rejected before domain mutation; security metric increments | remote fuzz result not recorded |
| request replay | resend identical request ID, sequence, and normalized payload inside cache lifetime | cached result is returned without repeating the action | two-client result not recorded |
| request ID content mismatch | reuse request ID and sequence with any changed normalized field | request is rejected as reuse; no domain action runs | remote fuzz result not recorded |
| sequence replay and gap | send an old sequence and a future sequence | both are rejected and the server supplies the expected next sequence | remote fuzz result not recorded |
| server-wide remote flood | many clients exhaust the server token bucket | excess requests are shed without unbounded tasks, persistence writes, or memory growth | load result not recorded |
| per-action flood | one client floods invite, offer, confirm, bootstrap, and sync separately | each action bucket limits work; legitimate unrelated players remain responsive | load result not recorded |
| target invite flood | many senders invite one target | target cooldown and single-trade state bound notifications and reservations | load result not recorded |
| expired invite/trade action | delay the expiry task, then respond, offer, or confirm after `expires_at` | synchronous expiry check closes the trade and rejects the action | scheduler fault result not recorded |
| telemetry overload | generate more alerts/audits than worker caps | correctness continues; telemetry drop metric increases; worker counts remain bounded | load result not recorded |
| recovery overload | expose more old transactions than recovery limits | only configured workers start; remaining indexed work is available to later sweeps | load result not recorded |
| player leaves before commit | disconnect either participant before both confirmations | live trade closes and inventories do not change | two-client result not recorded |
| player leaves during commit | disconnect after durable `committing` | recovery continues from persisted state and eventually finalizes exactly once | two-client result not recorded |
| shutdown with pending work | close server during prepare, abort, apply, and finalization in separate runs | shutdown stays bounded; unresolved durable state remains recoverable on another server | live result not recorded |
| messaging unavailable | fail recovery and alert publication while datastores work | correctness remains datastore-driven; periodic sweep or join recovery converges | live result not recorded |
| datastore budget exhaustion | force reads/writes below required budget | requests fail closed, queues/workers remain bounded, and no lease or transaction invariant is bypassed | soak result not recorded |

## Studio and multi-server procedure

1. Publish a dedicated test place with isolated datastore and messaging names.
2. Enable Studio access to api services for that test place only.
3. For an upgrade test, retire all legacy writer servers before permitting the new index migration to complete.
4. Use at least two test accounts that satisfy `minimum_account_age_days`.
5. Grant items through the server API without caller-supplied item IDs. Save every returned item ID.
6. Bootstrap both clients and record the issued session and next sequence.
7. Run the clean trade once, then exercise denial, replay, expiry, capacity, and collision cases.
8. For crash cases, instrument a test-only fault point after the exact durable write named in the matrix. Do not approximate the fault by stopping at an unknown time.
9. For coordinator fencing and lease contention, use two live servers pointed at the isolated test stores and retain both server logs.
10. After each run, inspect both inventories, the transaction or mutation operation record, its open-index shard when applicable, receipts, pending markers, metrics, and durable alerts.
11. Repeat crash and response-loss cases enough times to cover both operation orderings; distributed timing tests are not proven by one successful run.

## Soak and operational checks

- Run `run_lease_load_test` with the intended maximum players and a conservative update budget.
- Sustain valid trading plus malformed remote traffic for at least the expected server lifetime.
- Track datastore retries, consistent-read failures, lease refresh failures, remote rejections, commit recovery, telemetry drops, worker counts, memory, and network output.
- Repeat join/leave cycles and verify sessions, reservations, commit watchers, and player references are released.
- Fill inventories near the maximum and vary net outgoing/incoming counts.
- Hold transactions in every nonterminal state long enough to cross sweep and coordinator-lease expiry thresholds.
- Verify a full or unavailable open-index shard stops new mapped commits without losing prior entries.
- Seed the legacy index and an unindexed nonterminal `tx_` key, then verify the startup migration drains and reindexes them before its durable completion marker appears.
- Confirm that the game presents pending/recovery states honestly and never tells a client that an item transfer completed before terminal finalization.

## Release gate

A release candidate is ready for review only when:

- built-in verification passes with no disabled checks;
- every applicable matrix row has attached evidence for the exact code/configuration;
- no test violates conservation, capacity, collision, fencing, or decision-before-effects invariants;
- unresolved fault cases remain indexed and recoverable;
- overload tests remain within datastore, memory, task, and network budgets;
- migration and rollback procedures have been tested against backups;
- remaining limitations are accepted by the experience owner and reflected in player-facing behavior.
