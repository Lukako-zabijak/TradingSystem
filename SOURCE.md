# Source traceability

This repository is a clean-room modular refactor of the single 4,436-line trading server supplied with the application.

- Source SHA-256: `8ac458a7a48afa5d5e4df625a6d2a6aa0c5dd3487a193388e2d0bf1c4213f763`
- Preserved client protocol version: `1`
- Current inventory schema version: `3` (versions `1` and `2` migrate on read)
- Current mutation schema version: `1`
- Current transaction schema version: `3` (versions `1` and `2` migrate on read)

The refactor keeps the original external remote names, request actions, payload shapes, server bindable API names, existing DataStore names, and recovery topic names. Internal responsibilities were moved into explicit modules. Transaction schema `3` adds monotonic abort/commit decisions and coordinator fencing; inventory schema `3` adds pending-mutation recovery. Durable mutation operations, receipts, and uncached read-back reconcile ambiguous trusted inventory writes.
