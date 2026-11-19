# Admin Transfer Design

## Overview

This design proposes a two-step admin handover mechanism for the KeyLease Registry contract. The current admin can propose a new admin, and the new admin must explicitly accept the handover with their own authorization. This prevents accidental or compromised proposals from silently moving control of the contract.

## Key Concepts

### PendingAdmin

A `PendingAdmin` struct is stored in instance storage and contains:

- `new_admin`: The address that must authorize `accept_admin`
- `expires_at`: A ledger timestamp after which the proposal is stale and rejected

Proposals expire after `ADMIN_TRANSFER_EXPIRY_SECS` (7 days) to prevent forgotten handovers from being accepted indefinitely.

### Events

Three events track the admin transfer lifecycle:

- `AdminTransferProposed`: Emitted when the current admin proposes a handover
- `AdminTransferAccepted`: Emitted when the new admin accepts and becomes admin
- `AdminTransferCancelled`: Emitted when the current admin withdraws a pending handover

### API Entry Points

| Function | Description |
|---|---|
| `propose_admin(admin, new_admin)` | Current admin proposes a handover. Requires admin auth. A new proposal replaces any still-pending one. |
| `accept_admin(new_admin)` | New admin accepts the handover. Requires auth from `new_admin`. Proposal must exist, name `new_admin`, and not be expired. |
| `cancel_admin_transfer(admin)` | Current admin withdraws the pending handover. Keeps the current admin role. Requires admin auth. |
| `get_pending_admin()` | View function returning the pending admin, if any. |
| `migrate(to_version)` | Migrate storage schema version. Admin only. |

### Error Codes

- 17: `AdminTransferNotPending` — called when there is no pending proposal
- 18: `AdminTransferExpired` — the pending proposal has expired
- 19: `AdminUnchanged` — the proposed admin is already the current admin
- 20: `SchemaMigrationRequired` — storage schema migration needed before other mutations

## Security Model

- The current admin remains in control until the new admin accepts
- A new proposal always replaces any still-pending proposal
- Expired proposals are discarded and cannot be accepted
- The current admin can always cancel a live proposal
- Stale proposals (older than 7 days) cannot be accepted

## Migration Path

Existing deployments without schema versioning (schema version 0) must call `migrate(1)` after upgrading to this Wasm before other state mutations are allowed. The `migrate` entry point is the only mutation permitted while the stored schema does not match this Wasm.