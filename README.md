# HireSettle — Recruitment Fee Settlement Contract

A [Soroban](https://soroban.stellar.org/) smart contract deployed on Stellar for managing recruitment fee settlements through milestone-based escrow payments. Built with `#![no_std]` Rust and the Soroban SDK.

---

## Overview

HireSettle governs the relationship between a **hiring company** and a **recruiter** by locking the total agreed fee in an escrow wallet at engagement creation. As the recruiter delivers on each milestone (placement, 30-day retention, 90-day retention, etc.), the company confirms the deliverable, releasing a proportional payment from escrow. If disputes arise, an M-of-N arbiter panel votes to resolve them. The contract handles the entire lifecycle: creation, proof submission, confirmation, dispute, replacement, early exit, cancellation, and expiry.

### Core Concepts

- **Engagement** — A single recruitment contract identified by a unique string ID. Stores the company, recruiter, arbiters, token, total fee, milestone list, and lifecycle status.
- **Milestone** — A discrete payment trigger with a name, payment percentage, type (`Placement` or `Retention`), time-gate ledger, proof hash, and status (`Locked` → `Pending` → `ProofSubmitted` → `Confirmed` / `Disputed` → `Resolved`).
- **Escrow** — Funds are transferred from the company to the contract at creation. Milestone payments release proportional amounts to the recruiter (minus platform fees). Remaining escrow is refunded on cancellation or expiry.
- **Dispute Resolution** — The company raises a dispute on a proof-submitted milestone. Arbiters vote approve/reject; if `approve_votes >= quorum`, payment is released; if `reject_votes > arbiters.len() - quorum`, the proof is cleared and the milestone returns to `Pending`.

---

## Key Features

| Feature | Description |
|---|---|
| **Milestone-based escrow** | Funds locked at creation; released per-milestone on confirmation. Percentages sum to 100%. |
| **Placement & Retention types** | Placement starts `Pending`; retention starts `Locked` and requires a ledger time-gate to elapse. |
| **Multi-arbiter disputes** | M-of-N arbiter voting to resolve disputes. Approve releases payment; reject resets milestone. |
| **Co-recruiter split** | Optional `co_recruiter` address with configurable basis-point split for shared-fee engagements. |
| **Proof cooldown** | Configurable minimum ledger gap between proof resubmissions (default ~4 hours). |
| **Admin configuration** | Platform fee (up to 5%), token allowlist, max milestones, max retention days, inactivity timeout, confirm/dispute windows, arbiter fee (up to 2%), proof hash length, ledgers-per-day, storage TTL, upgrade lock duration. |
| **Auto / force confirm** | If the company does not act within the confirm window (~5 days), any address can force-confirm. |
| **Batch confirmation** | Confirm multiple milestones atomically in a single transaction. |
| **Recruiter early exit** | Recruiter requests exit; company accepts (refunds remaining escrow) or rejects (returns to Active). |
| **Candidate replacement** | Company requests replacement, resetting milestones and adjusting retention timers. |
| **Engagement expiry** | Permissionless keeper function — expires after an inactivity timeout (~60 days), refunds company. |
| **Amendment proposals** | Company or recruiter proposes a `payment_percent` change; the other party must accept within a TTL. Amendment history is logged (capped at 20 entries). |
| **Arbiter succession** | Arbiters can nominate and claim successors for their slots. |
| **Contract upgrade** | Admin proposes a WASM upgrade with a mandatory time-lock (~1 day); execution is permissionless after the lock. |
| **Admin renouncement** | Admin can permanently renounce, making the contract immutable. All admin-gated functions fail after renouncement. |
| **Contract PDF attestation** | Optional `contract_pdf_hash` (e.g. SHA-256 of the signed PDF) stored at creation for audit trail. |
| **Engagement listing** | Per-company paginated engagement ID list (issue #35) and global engagement counter (issue #34). |
| **Unlock progress query** | `get_unlock_progress()` returns `(unlocked_count, total)` — how many milestones are past `Locked` status. |

---

## Data Types

### `Engagement`
The full on-chain record:
- `id`, `company`, `recruiter`, `arbiters`, `quorum`, `token`
- `total_amount`, `released_amount`, `job_title`
- `metadata_hash` (optional IPFS CID), `contract_pdf_hash` (optional attestation hash)
- `created_at_ledger`, `last_activity_ledger`
- `milestones` (Vec<Milestone>), `status`, `co_recruiter`, `recruiter_split_bps`

### `Milestone`
- `name`, `payment_percent`, `kind` (Placement | Retention), `valid_after_ledger`
- `proof_hash`, `status` (Locked | Pending | ProofSubmitted | Confirmed | Disputed | Resolved), `proof_submitted_at`

### `EngagementConfig`
Passed at creation to stay within Soroban's 10-parameter limit:
- `metadata_hash` (Option<String>), `contract_pdf_hash` (Option<String>)
- `co_recruiter` (Option<Address>), `recruiter_split_bps` (u32)

### `EngagementStatus`
`Active` → `Completed` | `Cancelled` | `Expired` | `ReplacementRequested` | `ExitRequested`

---

## Public Function Reference

### Admin
`init`, `set_platform_fee`, `set_version`, `set_min_amount`, `pause`, `unpause`, `nominate_admin`, `claim_admin`, `renounce_admin`, `set_proof_cooldown`, `set_ledgers_per_day`, `set_max_retention_days`, `set_max_milestones`, `set_inactivity_timeout_ledgers`, `set_storage_ttl_extend_to`, `set_confirm_window`, `set_dispute_window`, `set_max_proof_hash_length`, `set_arbiter_fee`, `set_amendment_ttl`, `set_upgrade_lock_duration`, `propose_upgrade`, `add_allowed_token`, `remove_allowed_token`, `set_token_allowlist_enabled`

### Engagement Lifecycle
`create_engagement`, `unlock_milestone`, `submit_proof`, `confirm_milestone`, `batch_confirm_milestones`, `force_confirm_milestone`, `raise_dispute`, `cast_arbiter_vote`, `request_replacement`, `cancel_engagement`, `top_up_escrow`, `request_early_exit`, `accept_early_exit`, `reject_early_exit`, `expire_engagement`

### Amendments
`propose_amendment`, `accept_amendment`, `reject_amendment`

### Arbiter Succession
`nominate_arbiter_successor`, `claim_arbiter`

### Read-Only Queries
`get_engagement`, `get_engagement_summary`, `get_milestone`, `get_escrow_balance`, `get_total_released`, `get_metadata_hash`, `get_contract_pdf_hash`, `get_version`, `get_min_amount`, `get_platform_fee`, `get_pending_admin`, `get_amendment_ttl`, `get_amendment_log`, `get_pending_amendment`, `get_arbiter_votes`, `get_dispute_reason`, `get_replacement_reason`, `get_replacement_count`, `get_engagement_count`, `get_company_engagement_count`, `get_engagements_by_company`, `is_milestone_unlockable`, `ledgers_until_unlock`, `get_estimated_unlock_seconds`, `get_active_dispute_count`, `get_is_engagement_complete`, `get_unlock_progress`, `is_paused`, `get_ledgers_per_day`, `get_max_retention_days`, `get_max_milestones`, `get_inactivity_timeout_ledgers`, `get_storage_ttl_extend_to`, `get_confirm_window`, `get_dispute_window`, `get_max_proof_hash_length`, `get_arbiter_fee`, `get_upgrade_lock_duration`, `get_allowed_tokens`

---

## Events

The contract emits Soroban events for all state transitions: `engagement_created`, `milestone_unlocked`, `proof_submitted`, `proof_resubmitted`, `milestone_confirmed`, `engagement_completed`, `dispute_raised`, `arbiter_voted`, `dispute_resolved`, `replacement_requested`, `engagement_cancelled`, `early_exit_requested`, `early_exit_accepted`, `early_exit_rejected`, `engagement_expired`, `escrow_topped_up`, `amendment_proposed`, `amendment_accepted`, `amendment_rejected`, `platform_fee_collected`, `upgrade_proposed`, `upgrade_executed`, and admin configuration events.

---

## Project Structure

```
hiresettle-contract-1/
├── Cargo.toml              # Workspace root
├── contracts/
│   └── hiresettle/
│       ├── Cargo.toml      # Contract crate config
│       ├── Makefile        # Build helpers
│       └── src/
│           ├── lib.rs      # Main contract logic (~3500 lines)
│           └── test.rs     # Unit tests (~2000 lines)
├── TODO.md                 # Task tracking
└── README.md               # This file
```

---

## Testing

Run the full test suite from the contract directory:

```bash
cd contracts/hiresettle && cargo test
```

Tests cover creation, proof submission, confirmation, disputes, arbiter voting, replacement flow, early exit, amendments, batch confirmations, auto-confirm, expiry, admin configuration, and edge cases for all validation rules.

---

## Usage Example

```rust
// 1. Init contract
HireSettleContract::init(env, admin);

// 2. Create engagement
let config = EngagementConfig {
    metadata_hash: Some(String::from_str(&env, "Qm...")),
    co_recruiter: None,
    recruiter_split_bps: 10_000,
    contract_pdf_hash: Some(String::from_str(&env, "sha256:abc123...")),
};
HireSettleContract::create_engagement(
    env, "engagement-1", company, recruiter,
    arbiter_setup, token, 100_000_000, "Senior Engineer",
    milestones, retention_days, config
);

// 3. Recruiter unlocks & submits proof
HireSettleContract::unlock_milestone(env, "engagement-1", 1);
HireSettleContract::submit_proof(env, recruiter, "engagement-1", 1, "ipfs://QmProof...");

// 4. Company confirms → payment released
HireSettleContract::confirm_milestone(env, company, "engagement-1", 1);
```

---

## License

MIT

