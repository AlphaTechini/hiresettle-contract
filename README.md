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

### `EngagementSummary`

```rust
pub struct EngagementSummary {
    pub id: String,                  // unique engagement identifier
    pub job_title: String,           // short title set at creation
    pub company: Address,
    pub recruiter: Address,
    pub total_amount: i128,          // total fee locked (stroops)
    pub released_amount: i128,       // amount paid out so far
    pub status: EngagementStatus,
    pub milestone_count: u32,        // total milestones (does not change)
    pub created_at_ledger: u32,
}
```

Lightweight read-only view returned by `get_engagement_summary`, omitting the milestone vector for efficient dashboard listing.

### `AmendmentEntry`

```rust
pub struct AmendmentEntry {
    pub proposer: Address,           // company or recruiter who proposed
    pub old_payment_percent: u32,
    pub new_payment_percent: u32,
    pub ledger: u32,                 // ledger when the amendment was accepted
}
```

History entry recorded when a milestone payment-percent amendment is accepted.

### `AmendmentProposal`

```rust
pub struct AmendmentProposal {
    pub proposer: Address,
    pub new_payment_percent: u32,
    pub proposed_at_ledger: u32,
    pub expires_at_ledger: u32,      // proposal TTL; expires if not accepted in time
}
```

A pending milestone amendment proposal; either party may propose and the other must accept before expiry.

### `ArbiterSetup`

```rust
pub struct ArbiterSetup {
    pub arbiters: Vec<Address>,      // ordered list eligible to vote on disputes
    pub quorum: u32,                 // M-of-N votes required to resolve
}
```

Bundled argument passed to `create_engagement` to configure the arbitration panel (keeps parameter count within Soroban's 10-arg limit).

### `ArbiterVoteRecord`

```rust
pub struct ArbiterVoteRecord {
    pub approve_votes: u32,
    pub reject_votes: u32,
    pub voted: Vec<Address>,         // prevents double-voting
}
```

Per-dispute vote tally stored on-chain until the dispute resolves (approve or reject quorum is reached).

### `ArbiterVoteCounts`

```rust
pub struct ArbiterVoteCounts {
    pub approve_votes: u32,
    pub reject_votes: u32,
}
```

Returned by `get_arbiter_votes` — lightweight view of the current tally without the voter list.

### `ArbiterNomination`

```rust
pub struct ArbiterNomination {
    pub current: Address,            // nominating arbiter
    pub nominee: Address,            // successor
}
```

Stored under `DataKey::PendingArbiter` during arbiter succession; the nominee calls `claim_arbiter` to finalise.

### `UpgradeProposal`

```rust
pub struct UpgradeProposal {
    pub new_wasm_hash: BytesN<32>,   // new contract WASM hash
    pub execute_after_ledger: u32,   // earliest ledger at which execution is allowed
}
```

Pending contract WASM upgrade proposal; subject to an admin-configurable time-lock (default 17,280 ledgers ≈ 1 day).

### `PlatformFee`

```rust
pub struct PlatformFee {
    pub bps: u32,                    // fee in basis points (max 500 = 5%)
    pub treasury: Address,           // fee recipient
}
```

Platform fee deducted from each milestone payment before release to the recruiter.

### `DataKey`

```rust
pub enum DataKey {
    Engagement(String),              // full engagement record by ID (persistent)
    Admin,                           // current admin address (instance)
    PendingArbiter(String),          // pending arbiter succession nomination
    PlatformFee,                     // bps + treasury config (persistent)
    Paused,                          // pause-guard bool (persistent)
    PendingAdmin,                    // nominated admin successor (persistent)
    ProofCooldown,                   // min ledgers between resubmissions (instance)
    LastProofAt(String, u32),        // ledger of last proof submission
    ArbiterVotes(String, u32),       // running vote tally for a dispute
    AmendmentProposal(String, u32),  // active amendment proposal
    AmendmentLog(String, u32),       // amendment history entries
    AmendmentTTL,                    // proposal expiry duration (persistent)
    // … additional keys for counts, allowlist, timeouts, etc.
}
```

Contract storage key space enumerating all persistent and instance-stored values. Instance keys reset between transactions; persistent keys survive across ledgers.

---

## Amendments

Either the company or the recruiter may propose a change to a milestone's `payment_percent`. Amendments are scoped to a single milestone per proposal and require explicit acceptance from the counterparty before taking effect.

### Propose → Accept / Reject Flow

1. **Propose** — `propose_amendment(proposer, engagement_id, milestone_index, new_payment_percent)`
   - Caller must be the engagement's `company` or `recruiter`; must sign the transaction.
   - `new_payment_percent` is validated to be within 0–100 (inclusive).
   - A new proposal overwrites any existing pending proposal for the same milestone (only one pending per milestone at a time).
   - Emits an `amendment_proposed` event.
   - The pending state is stored as an [`AmendmentProposal`](#amendmentproposal) struct under `DataKey::AmendmentProposal(engagement_id, milestone_index)`.

2. **Accept** — `accept_amendment(acceptor, engagement_id, milestone_index)`
   - Caller must be the *other* party (the one who did **not** propose). A proposer cannot accept their own proposal.
   - The milestone's `payment_percent` is updated to the proposed value immediately.
   - An [`AmendmentEntry`](#amendmententry) is appended to the milestone's amendment log (see below) recording the old/new percentages, the proposer, and the acceptance ledger.
   - The pending proposal is cleared from storage.
   - Emits an `amendment_accepted` event.

3. **Reject** — `reject_amendment(rejector, engagement_id, milestone_index)`
   - Caller must be the *other* party (the one who did **not** propose). A proposer cannot reject their own proposal.
   - The pending proposal is cleared from storage without any change to the milestone.
   - Emits an `amendment_rejected` event with reason `declined`.

### TTL and Expiry

Every proposal carries an expiry ledger computed as `proposed_at_ledger + amendment_ttl` (see [`AmendmentProposal.expires_at_ledger`](#amendmentproposal)).

- The default TTL is **17,280 ledgers** (≈ 1 day at 5 s/ledger).
- Admin can change the default globally via `set_amendment_ttl(ledgers)`; the current value is queried with `get_amendment_ttl()`.
- If the current ledger exceeds `expires_at_ledger`, the proposal is considered *expired*:
  - Calling `accept_amendment` on an expired proposal clears it, emits an `amendment_rejected` event with reason `expired`, and panics with `amendment_expired`.
  - `get_pending_amendment` automatically treats expired proposals as non-existent and returns `None`.
- Expired proposals do **not** auto-clean from storage on ledger tick; they are lazily cleared on the next `accept_amendment`, `reject_amendment`, or overwritten by the next `propose_amendment` for the same milestone.

### What an Amendment Can Change

Only one field is mutable via the amendment system:

| Field | Type | Description |
|---|---|---|
| `milestone.payment_percent` | `u32` | Percentage of `total_amount` released when the milestone confirms. Must be 0–100 inclusive. |

An amendment does **not** change the total escrow, milestone status, proof hash, retention time-gates, arbiter configuration, or any other engagement field. Percentage-sum validation across all milestones is not re-enforced at amendment time; integrators are expected to ensure the combined set across all milestones still sums to 100 after applying accepted amendments.

### Amendment History (Log)

Each time an amendment is accepted, an [`AmendmentEntry`](#amendmententry) is appended to the per-milestone log:

- Accessible via `get_amendment_log(engagement_id, milestone_index)` which returns entries in chronological order (oldest first).
- The log is **FIFO-capped at 20 entries per milestone** — once the cap is reached, the oldest entry is evicted on the next accepted amendment.
- The pending proposal itself is not part of the log until it is accepted; use `get_pending_amendment` to inspect a live proposal.

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

### Create a test engagement via CLI

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- create_engagement \
  --engagement_id "ENG-TEST-001" \
  --company <COMPANY_ADDRESS> \
  --recruiter <RECRUITER_ADDRESS> \
  --arbiter_setup '{"arbiters":["<ARBITER_ADDRESS>"],"quorum":1}' \
  --token <USDC_SAC_ADDRESS> \
  --total_amount 5000000000 \
  --job_title "Senior Engineer" \
  --milestones '[...]' \
  --retention_days '[30, 90]'
```

> USDC SAC on Testnet: `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA`

---

## Security Considerations

- **Authorization**: Every state-changing function calls `require_auth()`. Recruiters cannot confirm their own milestones. Companies cannot cast arbiter votes.
- **Multi-arbiter quorum**: Disputes require M-of-N arbiter votes to resolve. A single arbiter cannot unilaterally release or withhold payment — both approval and rejection require a configurable quorum. Duplicate votes from the same arbiter are rejected on-chain.
- **Arbiter fee cap**: The arbiter fee is capped at 200 bps (2%) to prevent excessive deduction from recruiter payouts on dispute approval.
- **Retention double-check**: `confirm_milestone()` re-verifies `valid_after_ledger` even if `unlock_milestone()` was called, preventing a company from confirming a retention milestone before the window truly ends.
- **Replacement fee fairness**: The Placement tranche paid to the recruiter is non-refundable. Only unreleased amounts are frozen. This is explicit in the contract and documented clearly so both parties understand the terms at engagement creation.
- **Ledger drift**: The 5s/ledger assumption is approximate. Stellar's actual ledger time may vary slightly. The contract uses ledger sequence numbers — not timestamps — so the unlock is purely count-based. Production deployments should account for ~±5% drift in real-world retention windows.
- **Upgrade time-lock**: Contract upgrades require a configurable time-lock (default 17,280 ledgers ≈ 1 day) between proposal and execution, giving stakeholders time to review before changes take effect.

---

## Roadmap

- [x] Core escrow + milestone logic
- [x] Time-gated retention milestones (ledger-based unlock)
- [x] Replacement clause with clock reset
- [x] Dispute resolution via arbiter
- [x] Flexible milestone structure (2-milestone 50/50, 3-milestone, custom)
- [x] 11 unit tests
- [ ] Multi-candidate engagements (multiple positions, one company-recruiter pair)
- [ ] Partial payout on replacement (configurable replacement fee)
- [x] Contract upgrade mechanism
- [ ] Mainnet deployment

---

## License

MIT

