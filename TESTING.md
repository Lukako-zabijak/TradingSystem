# Testing checklist

## Automated checks included

The startup verification module tests:

- successful two-party commit;
- rollback when the second prepare fails;
- completion of a partially applied committing transaction;
- rejection of unknown client payload fields;
- deterministic offer validation and normalization.

The bootstrap server refuses to connect remotes when these checks fail. Set the server script attribute `disable_self_tests` to `true` only for targeted diagnosis.

## Studio integration test

Use a separate published test place with Studio API access enabled.

1. Join with two clients whose account ages satisfy the configured minimum.
2. Grant each client at least one unique tradable item through the server API.
3. Bootstrap both clients and verify their server-issued session and sequence.
4. Invite, accept, set both offers, and confirm the same trade revision.
5. Verify both inventories increment revision once and contain the opposite item.
6. Retry the same request ID and sequence; verify the cached result is returned without another mutation.
7. Send an extra payload field, a duplicate item ID, a stale revision, and an invalid session; verify each is rejected.
8. Disconnect one client before confirmation; verify the trade closes without changing inventory.
9. Disconnect during commit or stop the server after the first apply; rejoin and verify recovery completes exactly once.
10. Attempt to join the same user from another server while the first lease is active; verify the second session is refused.

## Long-session checks

- Run `run_lease_load_test` with the expected player count and DataStore update budget.
- Inspect `get_metrics` for retry growth, lease failures, remote rejections, and incomplete commits.
- Inspect `get_durable_alerts` after forced DataStore and MessagingService failures.
- Repeat join/leave and trade cycles while watching server memory and connection counts.
