# HireSettle — Contract Repo

> **Milestone-based recruiter fee escrow on Stellar Soroban**

HireSettle locks a recruiter's fee in a Soroban smart contract when a company opens a role. The fee releases automatically in tranches as the candidate clears each hiring milestone — placement, 30-day retention, and 90-day retention. No invoice chasing, no refund disputes over bad hires, no trust required between strangers.

This is **Repo 1 of 3** in the HireSettle project:

| Repo | Description |
|------|-------------|
| `hiresettle-contract` ← you are here | Soroban smart contract (Rust) |
| `hiresettle-backend` | NestJS REST API + event poller |
| `hiresettle-frontend` | Next.js + Freighter wallet UI |

---

## Table of Contents

- [How It Works](#how-it-works)
- [What Makes This Different from ChainSettle](#what-makes-this-different-from-chainsetttle)
- [Architecture](#architecture)
- [Data Structures](#data-structures)
- [Milestone State Machine](#milestone-state-machine)
- [Contract Functions](#contract-functions)
- [Time-Gated Retention Logic](#time-gated-retention-logic)
- [Replacement Clause](#replacement-clause)
- [Events](#events)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Setup & Installation](#setup--installation)
- [Running Tests](#running-tests)
- [Building](#building)
- [Deploying to Testnet](#deploying-to-testnet)
- [Security Considerations](#security-considerations)
- [Roadmap](#roadmap)

---

## How It Works

```
Company posts role → fee locked in Soroban escrow (USDC)
         ↓
Recruiter finds candidate → submits offer letter proof (Milestone 1 — Placement)
         ↓
Company confirms → 30% fee released to recruiter
         ↓
30 days elapse → ledger timer unlocks Milestone 2
         ↓
Recruiter submits HR confirmation → Company confirms → 40% released
         ↓
90 days elapse → ledger timer unlocks Milestone 3
         ↓
Recruiter submits payroll proof → Company confirms → 30% released → Complete ✓

If candidate leaves early → Company calls request_replacement()
  → Placement milestone resets, retention clocks restart
  → Recruiter must find a new candidate to claim remaining balance
```

---

## What Makes This Different from ChainSettle

HireSettle extends the ChainSettle escrow pattern with three features specific to recruitment:

### 1. Time-Gated Retention Milestones

Retention milestones have a `valid_after_ledger` — a specific Stellar ledger sequence number before which the milestone cannot be confirmed. The contract calculates this from `retention_days` at creation time:

```
valid_after_ledger = created_at_ledger + (retention_days × 17_280)
```

17,280 is the approximate number of Stellar ledgers per day (86,400 seconds ÷ 5 seconds per ledger).

A Locked milestone progresses to Pending only after `unlock_milestone()` is called once the ledger timestamp has passed. The backend cron job calls this automatically at day 30 and day 90, but anyone can trigger it.

### 2. Replacement Clause

If a placed candidate leaves before 90-day retention is confirmed, the company calls `request_replacement()`. This:
- Resets the Placement milestone to Pending (recruiter must find a new candidate)
- Restarts all uncompleted retention clocks from the current ledger
- Freezes the remaining escrow balance until the replacement placement is confirmed
- Keeps the Placement fee already paid — the recruiter earned that tranche

### 3. `MilestoneKind` Enum

Each milestone declares its kind: `Placement` (immediately available, recruiter submits offer proof) or `Retention` (time-locked, recruiter submits HR or payroll record). The contract enforces unlock rules based on kind, so Placement milestones can never accidentally be locked, and Retention milestones can never bypass the timer.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                HireSettle Contract (Soroban)                  │
│                                                              │
│  ┌───────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Engagement   │  │  Milestone State │  │  Time-Gated  │  │
│  │  Registry     │  │  Machine         │  │  Unlock      │  │
│  │  (Persistent) │  │  (per milestone) │  │  (ledger)    │  │
│  └───────────────┘  └──────────────────┘  └──────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Replacement Clause — resets placement + clocks       │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Roles

| Role | Address | Permissions |
|------|---------|-------------|
| **Company** | Locks USDC, confirms milestones, raises disputes, cancels, requests replacement | Primary actor |
| **Recruiter** | Submits proof documents at each milestone | `submit_proof` only |
| **Arbiter** | Votes to resolve disputes; quorum of arbiters must agree before a disputed milestone moves to `Resolved` or back to `Pending` | `cast_arbiter_vote` only |

---

## Data Structures

### `Milestone`

```rust
pub struct Milestone {
    pub name: String,                // e.g. "Candidate Placed", "30-Day Retention"
    pub payment_percent: u32,        // 0-100, all milestones must sum to 100
    pub kind: MilestoneKind,         // Placement | Retention
    pub valid_after_ledger: u32,     // for Retention: unlock ledger; for Placement: 0
    pub proof_hash: String,          // IPFS CID set by recruiter
    pub status: MilestoneStatus,     // see state machine below
}
```

### `Engagement`

```rust
pub struct Engagement {
    pub id: String,                  // unique company-defined ID e.g. "ENG-2026-001"
    pub company: Address,
    pub recruiter: Address,
    pub arbiters: Vec<Address>,      // ordered list of arbiters eligible to vote on disputes
    pub quorum: u32,                 // number of arbiter votes required to resolve (M-of-N)
    pub token: Address,              // USDC Stellar Asset Contract
    pub total_amount: i128,          // total fee locked (stroops)
    pub released_amount: i128,       // amount paid out so far
    pub job_title: String,           // short title (≤64 chars); full metadata off-chain
    pub created_at_ledger: u32,      // ledger when created (used for replacement clock reset)
    pub milestones: Vec<Milestone>,
    pub status: EngagementStatus,    // Active | Completed | Cancelled | ReplacementRequested
}
```

---

## Milestone State Machine

```
Locked ──── unlock_milestone() ──→ Pending
                                     │
                             submit_proof()
                                     │
                                     ↓
                              ProofSubmitted
                              /            \
              confirm_milestone()      raise_dispute(reason)
                    │                       │
                    ↓                       ↓
                Confirmed               Disputed
                                       /        \
                  cast_arbiter_vote(approve=true)  cast_arbiter_vote(approve=false)
                  [quorum reached]                 [rejection quorum reached]
                              │                      │
                              ↓                      ↓
                           Resolved               Pending  ← recruiter resubmits
```

Placement milestones start as **Pending** (immediately available).
Retention milestones start as **Locked** and require `unlock_milestone()` after the time window.

---

## Contract Functions

### `init(admin: Address)`
Initialises the contract. Called once by the deployer.

### `create_engagement(...) → String`
Creates a new engagement and transfers `total_amount` USDC from the company into escrow. Validates that milestone percentages sum to 100. Computes `valid_after_ledger` for each Retention milestone from `retention_days`.

```
Parameters:
  engagement_id   String           — unique ID
  company         Address          — funds source + milestone approver
  recruiter       Address          — payment recipient
  arbiter_setup   ArbiterSetup     — { arbiters: Vec<Address>, quorum: u32 }
  token           Address          — USDC SAC address
  total_amount    i128             — total fee in stroops
  job_title       String           — short job description
  milestones      Vec<Milestone>   — ordered list
  retention_days  Vec<u32>         — one day count per Retention milestone
  metadata_hash   Option<String>   — optional IPFS CID for full job description
```

### `unlock_milestone(engagement_id, milestone_index)`
Moves a Locked Retention milestone to Pending once `current_ledger >= valid_after_ledger`. Anyone can call this — it's permissionless. The backend calls it automatically at the retention window, but it can be triggered manually.

### `submit_proof(recruiter, engagement_id, milestone_index, proof_hash)`
Recruiter submits an IPFS hash or URL as proof for a Pending milestone.
- Placement: signed offer letter CID
- 30-day: HR confirmation email or payroll record
- 90-day: payroll record or employment certificate

### `confirm_milestone(company, engagement_id, milestone_index)`
Company confirms a ProofSubmitted milestone. Releases the milestone's payment % to the recruiter. For Retention milestones, double-checks that `current_ledger >= valid_after_ledger` as a safety guard.

### `raise_dispute(company, engagement_id, milestone_index, reason: String)`
Company disputes a ProofSubmitted milestone. Freezes it in Disputed status. The `reason` string (max 128 characters) is stored on-chain and queryable via `get_dispute_reason`.

### `cast_arbiter_vote(arbiter, engagement_id, milestone_index, approve: bool)`
Each arbiter casts a single vote on a Disputed milestone. The dispute resolves automatically once a quorum is reached:
- `approve_votes >= quorum` → payment released to recruiter, milestone moves to `Resolved`
- `reject_votes > total_arbiters - quorum` → proof cleared, milestone resets to `Pending` for the recruiter to resubmit

Duplicate votes from the same arbiter are rejected. Vote tallies are tracked on-chain per `(engagement_id, milestone_index)` and cleared once the dispute is resolved.

### `request_replacement(company, engagement_id)`
Company requests a replacement when a candidate leaves. Requires Placement to be already confirmed. Resets Placement milestone to Pending, restarts all unconfirmed retention clocks.

### `cancel_engagement(company, engagement_id)`
Cancels the engagement and refunds the full amount to the company. Only allowed before any milestones are confirmed.

### Upgradability

The contract supports admin-gated WASM upgrades behind a mandatory time-lock, so
there's always a review window between an upgrade being proposed and it taking effect.

#### `propose_upgrade(admin: Address, new_wasm_hash: BytesN<32>)`
Admin proposes a new contract WASM hash. The upgrade becomes executable after the
configured time-lock (`upgrade_lock_duration`, default **17,280 ledgers ≈ 1 day**)
has elapsed. Only one proposal can be pending at a time — calling this again
overwrites the existing proposal and resets its unlock ledger. Emits
`upgrade_proposed` with `(new_wasm_hash, execute_after_ledger)`.

#### `execute_upgrade()`
Applies the pending upgrade. **Permissionless** — anyone can call this once the
time-lock has elapsed; it doesn't require the admin's signature. Panics with
`"no pending upgrade"` if nothing was proposed, or `"UpgradeLockNotElapsed"` if
called before `execute_after_ledger`. Emits `upgrade_executed` and then swaps the
contract's WASM via `update_current_contract_wasm`.

#### `set_upgrade_lock_duration(admin: Address, ledgers: u32)`
Admin sets how many ledgers must elapse between a proposal and its execution.
Defaults to 17,280 (~1 day) if never called.

#### `get_upgrade_lock_duration() → u32`
Read-only. Returns the currently configured time-lock duration in ledgers.

### Read-only queries

| Function | Returns |
|---|---|
| `get_engagement(engagement_id)` | Full `Engagement` struct |
| `get_milestone(engagement_id, milestone_index)` | Single `Milestone` |
| `get_escrow_balance(engagement_id)` | i128 — USDC still locked |
| `is_milestone_unlockable(engagement_id, milestone_index)` | bool |
| `ledgers_until_unlock(engagement_id, milestone_index)` | u32 — ledgers remaining |

---

## Time-Gated Retention Logic

Stellar produces a new ledger approximately every 5 seconds. The contract converts `retention_days` to a ledger offset:

```rust
const LEDGERS_PER_DAY: u32 = 17_280; // 86400 ÷ 5
let unlock_ledger = current_ledger + (days * LEDGERS_PER_DAY);
```

The backend uses `ledgers_until_unlock()` to know exactly when to schedule the unlock notification. When the remaining ledgers reach 0, it calls `unlock_milestone()` on-chain.

The frontend displays the retention timer as a human-readable countdown:

```typescript
const daysRemaining = Math.ceil(ledgersRemaining / 17280);
```

---

## Replacement Clause

When `request_replacement()` is called:

1. The engagement status moves to `ReplacementRequested`
2. The Placement milestone resets to `Pending` (proof_hash cleared)
3. All Retention milestones that haven't been confirmed reset to `Locked`
4. Retention `valid_after_ledger` values are recalculated from the **current ledger** using the original day windows
5. The escrow balance is not changed — the remaining amount stays frozen

When the recruiter submits proof for the replacement Placement milestone, the engagement automatically returns to `Active`.

The Placement fee already released to the recruiter is **not clawed back** — that tranche was earned. Only the unreleased balance is frozen.

---

## Events

| Event name | Payload | When |
|---|---|---|
| `engagement_created` | `engagement_id` | New engagement created |
| `milestone_unlocked` | `(engagement_id, milestone_index)` | Retention milestone unlocked |
| `proof_submitted` | `(engagement_id, milestone_index)` | Proof submitted |
| `milestone_confirmed` | `(engagement_id, milestone_index, payment)` | Milestone confirmed, fee released |
| `dispute_raised` | `(engagement_id, milestone_index)` | Dispute opened |
| `arbiter_voted` | `(engagement_id, milestone_index, approve)` | Single arbiter casts a vote on a dispute |
| `dispute_resolved` | `(engagement_id, milestone_index, approved)` | Dispute resolved (quorum reached) |
| `replacement_requested` | `engagement_id` | Replacement requested |
| `engagement_cancelled` | `(engagement_id, refund_amount)` | Engagement cancelled |

---

## Project Structure

```
hiresettle-contract/
├── Cargo.toml                         ← Rust workspace config
├── Cargo.lock
├── .gitignore
├── README.md
└── contracts/
    └── hiresettle/
        ├── Cargo.toml                 ← Contract package config
        ├── Makefile                   ← Build / deploy shortcuts
        └── src/
            ├── lib.rs                 ← Full contract logic
            └── test.rs                ← 11 unit tests
```

---

## Prerequisites

```bash
# Rust + wasm32 target
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32v1-none

# Stellar CLI
cargo install --locked stellar-cli --features opt

# Testnet account
stellar keys generate --global my-account --network testnet
stellar keys fund my-account --network testnet
```

---

## Setup & Installation

```bash
git clone https://github.com/your-org/hiresettle-contract.git
cd hiresettle-contract
cargo check
```

---

## Running Tests

```bash
cargo test
cargo test -- --nocapture       # with output
cargo test test_full_engagement # single test
```

Expected:
```
running 11 tests
test test::test_cancel_engagement .......................... ok
test test::test_cannot_cancel_after_placement_confirmed .... ok
test test::test_cannot_unlock_before_window ................ ok
test test::test_create_engagement_invalid_percentages ...... ok
test test::test_create_engagement_success .................. ok
test test::test_full_engagement_lifecycle .................. ok
test test::test_ledgers_until_unlock ....................... ok
test test::test_raise_and_resolve_dispute_approve .......... ok
test test::test_raise_and_resolve_dispute_reject ........... ok
test test::test_request_replacement ........................ ok
test test::test_request_replacement_before_placement ....... ok
test test::test_two_milestone_engagement_50_50 ............. ok
```

---

## Building

```bash
make build      # → target/wasm32v1-none/release/hiresettle.wasm
make optimize   # → target/wasm32v1-none/release/hiresettle.optimized.wasm
```

---

## Deploying to Testnet

```bash
export STELLAR_ACCOUNT=my-account
make deploy-testnet
# → CXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Initialize after deployment

```bash
stellar contract invoke \
  --id <CONTRACT_ID> \
  --source my-account \
  --network testnet \
  -- init \
  --admin <YOUR_ADDRESS>
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
- [ ] Contract upgrade mechanism
- [ ] Mainnet deployment

---

## License

MIT
