# Contributing to HireSettle Contract

Thank you for your interest in contributing to the HireSettle contract! This repository runs a bounty-style "Stellar Wave" program — we welcome contributions from the community.

This guide documents the local development workflow, testing procedures, and pull request (PR) process so new contributors can get started quickly without reverse-engineering from the README.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Help](#getting-help)
- [Prerequisites](#prerequisites)
- [Local Setup](#local-setup)
- [Running Tests](#running-tests)
- [Formatting & Linting](#formatting--linting)
- [Building the Contract](#building-the-contract)
- [Pull Request Process](#pull-request-process)
- [CI Checks](#ci-checks)
- [Security](#security)
- [License](#license)

---

## Code of Conduct

By participating in this project, you agree to treat all contributors with respect. Report unacceptable behaviour to the maintainers.

---

## Getting Help

If you have questions about the contract, its design, or how to implement a feature, open a GitHub issue and tag the maintainers. For security concerns, see [SECURITY.md](file:///c:/Users/USER/downloads/drips/hiresettle-contract/SECURITY.md).

---

## Prerequisites

Before you begin, ensure the following tools are installed on your local machine:

| Tool | Description |
|------|-------------|
| **Rust** (latest stable) | The programming language used for the contract. Install via [rustup.rs](https://rustup.rs/). |
| **Cargo** | Rust's build system and package manager. Ships with Rust. |
| **Soroban CLI** | Stellar's smart contract CLI for building and deploying Soroban contracts. |
| **Stellar CLI** | The Stellar command-line interface for interacting with the network. |
| **Git** | Version control. |

Additionally, add the WASM target for Soroban builds:

```bash
rustup target add wasm32v1-none
```

---

## Local Setup

### 1. Fork & Clone the Repository

Fork the repo on GitHub, then clone your fork locally:

```bash
git clone https://github.com/<your-username>/hiresettle-contract.git
cd hiresettle-contract
```

Add the upstream remote to keep your fork in sync:

```bash
git remote add upstream https://github.com/TrustHire/hiresettle-contract.git
```

### 2. Navigate to the Contract Crate

The contract code lives in a sub-crate:

```bash
cd contracts/hiresettle
```

### 3. Verify Your Setup

Confirm that the code compiles without errors:

```bash
cargo check
```

Or use the Makefile shorthand:

```bash
make check
```

---

## Running Tests

The project uses `cargo test` with the Soroban SDK's `testutils` feature enabled.

### Run the Full Test Suite

From the contract directory (`contracts/hiresettle`):

```bash
cargo test
```

Or via Makefile:

```bash
make test
```

From the workspace root, you can also run:

```bash
cargo test -p hiresettle
```

### Run a Specific Test

To run a single test by name (or filter tests by pattern):

```bash
cargo test <test_name_or_pattern>
```

### Test Coverage

Tests cover creation, proof submission, confirmation, disputes, arbiter voting, replacement flow, early exit, amendments, batch confirmations, auto-confirm, expiry, admin configuration, and edge cases for all validation rules.

When adding a new feature or fixing a bug, include corresponding test cases in [test.rs](file:///c:/Users/USER/downloads/drips/hiresettle-contract/contracts/hiresettle/src/test.rs).

---

## Formatting & Linting

**All PRs must pass formatting and linting checks before submission.** Run these locally before pushing to save CI time.

### Formatting with `cargo fmt`

Format all Rust source code:

```bash
cargo fmt
```

Or via Makefile:

```bash
make fmt
```

To check formatting **without** writing changes (useful for CI-style verification):

```bash
cargo fmt --check
```

### Linting with `cargo clippy`

Run Clippy to catch common mistakes and enforce idiomatic Rust:

```bash
cargo clippy -- -D warnings
```

The `-D warnings` flag treats all Clippy warnings as errors. If you believe a lint is a false positive or should be suppressed, discuss it in your PR.

For the WASM target (mirrors what CI will run on the build artifact):

```bash
cargo clippy --target wasm32v1-none -- -D warnings
```

---

## Building the Contract

### Development Build

A standard debug build (fast compile, includes debug info):

```bash
cargo build
```

### Production Build (WASM)

For deployment, build and optimize the WASM binary using the Stellar CLI:

```bash
stellar contract build
stellar contract optimize --wasm target/wasm32v1-none/release/hiresettle.wasm
```

Or via Makefile:

```bash
make optimize
```

The optimized WASM will be at:

```
target/wasm32v1-none/release/hiresettle.optimized.wasm
```

---

## Pull Request Process

Follow these steps to submit a contribution:

### 1. Create a Feature Branch

From an up-to-date `main`:

```bash
git fetch upstream
git checkout main
git merge upstream/main
git checkout -b feature/<descriptive-name>
```

Use descriptive branch names like `feature/batch-confirm-sequential-fix` or `fix/amendment-expiry-logic`.

### 2. Make Your Changes

- Keep commits atomic and focused.
- Write clear, descriptive commit messages.
- If your change addresses an open issue, reference it in the commit message (e.g., `Fixes #123: ...`).

### 3. Run the Pre-Submission Checklist

Before opening a PR, **locally run all three**:

| Step | Command (from `contracts/hiresettle`) |
|------|----------------------------------------|
| ✅ Format | `cargo fmt` |
| ✅ Lint   | `cargo clippy -- -D warnings` |
| ✅ Test   | `cargo test` |

Fix any failures before pushing.

### 4. Push & Open the PR

Push your branch to your fork:

```bash
git push origin feature/<descriptive-name>
```

Then open a PR on GitHub against `TrustHire/hiresettle-contract:main`.

#### PR Template Checklist

In your PR description, confirm that you have:

- [ ] Read and followed this CONTRIBUTING guide.
- [ ] Run `cargo fmt` locally.
- [ ] Run `cargo clippy -- -D warnings` locally with no errors.
- [ ] Run `cargo test` locally with all tests passing.
- [ ] Added tests for new functionality or bug fixes.
- [ ] Updated relevant documentation (README, function doc comments, etc.) if applicable.
- [ ] Linked any related issues (e.g., `Closes #42`, `Fixes #71`).

### 5. Address Review Feedback

Maintainers will review your PR. Push additional commits to your branch to address feedback — do not rebase or force-push during review unless explicitly asked. Once approved, the PR will be squash-merged into `main`.

### 6. Claim Your Bounty (Stellar Wave Program)

For bounty-eligible contributions, follow the instructions posted in the Stellar Wave program to claim your reward after the PR is merged.

---

## CI Checks

When you open a PR, automated CI runs will validate your changes. **A PR must pass all required checks before it can be merged.**

The CI pipeline will verify the following (some may be enabled incrementally as the related CI configurations are rolled out):

| CI Check | What it does | Equivalent Local Command |
|----------|--------------|---------------------------|
| **Cargo Format Check** | Verifies `cargo fmt` has been applied. No formatting changes left. | `cargo fmt --check` |
| **Cargo Clippy** | Runs linting; errors on any warning. | `cargo clippy -- -D warnings` |
| **Cargo Test** | Executes the full test suite. | `cargo test` |
| **Cargo Build (WASM)** | Builds the contract for `wasm32v1-none`. | `stellar contract build` |
| **Security Audit** | (When implemented) Runs `cargo audit` for dependency vulnerabilities. | `cargo audit` |
| **WASM Optimization Check** | (When implemented) Verifies the optimized WASM can be produced deterministically. | `stellar contract optimize` |

If a CI check fails, click through to the run log, diagnose the issue, and push a fix to your PR branch. The checks will re-run automatically.

---

## Security

For security vulnerabilities, **do not open a public issue**. Follow the responsible disclosure process documented in [SECURITY.md](file:///c:/Users/USER/downloads/drips/hiresettle-contract/SECURITY.md).

---

## License

By contributing to this repository, you agree that your contributions will be licensed under the MIT License, as specified in [LICENSE](file:///c:/Users/USER/downloads/drips/hiresettle-contract/LICENSE).
