# Source traceability

This repository is a clean-room modular refactor of the single 4,436-line trading server supplied with the application.

- Source SHA-256: `8ac458a7a48afa5d5e4df625a6d2a6aa0c5dd3487a193388e2d0bf1c4213f763`
- Preserved client protocol version: `1`
- Preserved inventory schema version: `2`
- Preserved transaction schema version: `2`

The refactor keeps the original external remote names, request actions, payload shapes, server bindable API names, DataStore names, and recovery topic names. Internal responsibilities were moved into explicit modules and mutation receipts were retained for ambiguous-write reconciliation.
