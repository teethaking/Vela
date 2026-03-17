# Contributing to Vela

Thank you for your interest in contributing to Vela. This document covers everything you need to get involved. Vela has a unique architecture compared to most Soroban projects — the reporter system, signal aggregation, and SBT mechanics each have specific contribution patterns worth understanding before you start.

---

## Table of contents

- [Code of conduct](#code-of-conduct)
- [Ways to contribute](#ways-to-contribute)
- [Getting started](#getting-started)
- [Project structure](#project-structure)
- [Development workflow](#development-workflow)
- [Writing Soroban contracts](#writing-soroban-contracts)
- [Working with the reporter system](#working-with-the-reporter-system)
- [Writing the SDK](#writing-the-sdk)
- [Testing](#testing)
- [Pull request process](#pull-request-process)
- [Issue guidelines](#issue-guidelines)
- [Security vulnerabilities](#security-vulnerabilities)
- [Community](#community)

---

## Code of conduct

Vela follows the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) Code of Conduct. Report unacceptable behaviour via the repository's security policy email.

---

## Ways to contribute

**Code**
- Implement scoring model improvements
- Build new reporter adapters for ecosystem protocols
- Fix bugs and improve test coverage
- Optimise Soroban storage and compute usage

**Research**
- Propose and model scoring weight adjustments
- Research Sybil attack vectors and mitigations
- Analyse score gaming scenarios and countermeasures
- Model score decay rates and their effects on borrower behaviour

**Documentation**
- Improve lender integration guides
- Write tutorials for wallet owners and borrowers
- Document reporter contract standards

**Community**
- Answer questions in GitHub Discussions
- Review open pull requests
- Report bugs with clear reproduction steps

---

## Getting started

### Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Rust | `>=1.74` | Soroban contract development |
| `soroban-cli` | latest | Contract build, deploy, invoke |
| Node.js | `>=18` | Vela.js SDK and backend |
| pnpm | `>=8` | SDK package management |
| Docker | any | Local Stellar testnet |

### Install Rust and Soroban toolchain

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-unknown-unknown
cargo install --locked soroban-cli
```

### Clone and build

```bash
git clone https://github.com/your-org/vela
cd vela
cargo build
```

### Start a local testnet

```bash
docker run --rm -it \
  -p 8000:8000 \
  stellar/quickstart:latest \
  --testnet \
  --enable-soroban-rpc
```

### Run the full test suite

```bash
cargo test
```

---

## Project structure

```
vela/
├── contracts/
│   ├── signal-aggregator/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── storage.rs
│   │   │   ├── registry.rs
│   │   │   └── types.rs
│   │   └── Cargo.toml
│   ├── score-engine/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── model.rs
│   │   │   ├── normalise.rs
│   │   │   ├── decay.rs
│   │   │   └── types.rs
│   │   └── Cargo.toml
│   ├── score-token/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   └── types.rs
│   │   └── Cargo.toml
│   └── reporters/
│       ├── anchor-reporter/
│       ├── stream-reporter/
│       ├── pulsar-reporter/
│       ├── quake-reporter/
│       ├── wallet-age-reporter/
│       └── loan-reporter/
├── backend/
│   └── src/
├── frontend/
│   └── src/
├── sdk/
│   └── src/
├── docs/
├── CONTRIBUTING.md
├── README.md
└── LICENSE
```

---

## Development workflow

Vela uses a standard fork-and-branch workflow.

### 1. Fork and clone

```bash
git clone https://github.com/YOUR_USERNAME/vela
cd vela
git remote add upstream https://github.com/your-org/vela
```

### 2. Branch naming

```
feat/short-description        # new feature
fix/short-description         # bug fix
reporter/protocol-name        # new reporter adapter
scoring/short-description     # scoring model change
docs/short-description        # documentation only
test/short-description        # tests only
```

### 3. Commit conventions

```
[signal-aggregator] Add per-reporter signal cap enforcement
[score-engine] Adjust anchor consistency weight from 12% to 14%
[pulsar-reporter] Implement BillingExecuted event listener
[sdk] Add getScoreHistory() method
```

### 4. Sync with upstream

```bash
git fetch upstream
git rebase upstream/main
```

---

## Writing Soroban contracts

### Style conventions

- `snake_case` for all functions and variables, `PascalCase` for types
- Every public function must have a doc comment: purpose, parameters, panics, events emitted
- Use `VelaError` enum — never bare `panic!`
- Keep public contract functions thin, delegate logic to internal modules

### The scoring model is sensitive

Any change to `model.rs` in the `score-engine` contract — weights, normalisation logic, decay rate, score range — is a breaking change for existing lenders. Rules for scoring model changes:

- All weight adjustments require a governance issue opened for community discussion before a PR is submitted
- New signal categories require two maintainer approvals regardless of implementation quality
- Scoring model changes must include a migration impact analysis showing score distribution shift
- Every scoring model change must bump the `MODEL_VERSION` constant

### Error handling

All errors must be variants of `VelaError`:

```rust
#[contracterror]
#[derive(Copy, Clone, Debug, Eq, PartialEq, PartialOrd, Ord)]
#[repr(u32)]
pub enum VelaError {
    ReporterNotRegistered    = 1,
    ReporterAlreadyExists    = 2,
    SubjectHasNoHistory      = 3,
    ScoreTokenAlreadyMinted  = 4,
    SignalValueOutOfRange    = 5,
    EvidenceLedgerInFuture   = 6,
    Unauthorised             = 7,
    SignalCapExceeded         = 8,
    InvalidSignalCategory    = 9,
    ScoreNotFound            = 10,
}
```

### The ScoreToken has no transfer function

When contributing to `score-token`, never add a transfer or approve function under any circumstances. The non-transferability is a core protocol guarantee. A PR adding transfer functionality will be rejected immediately regardless of stated purpose.

---

## Working with the reporter system

Reporters are the most active contribution area for ecosystem developers. A reporter is a thin adapter contract that listens to events from a Stellar protocol and translates them into `report_signal()` calls on the `SignalAggregator`.

### Writing a new reporter

Every reporter must:

1. Be a Soroban contract in `/contracts/reporters/<name>-reporter/`
2. Implement the `Reporter` interface (defined in `contracts/reporters/reporter-interface/`)
3. Reference a specific `evidence_ledger` for every signal — the ledger where the underlying event occurred
4. Cap signal values to the 0–1_000_000 range before submission
5. Never submit the same `(subject, category, evidence_ledger)` combination twice
6. Include a `get_reporter_info()` view function returning name, category, and version

### Reporter registration

New reporters are not auto-registered. After implementing and testing, open a governance issue requesting registration. The proposal should include: what events the reporter listens to, how signal values are calculated, and the evidence_ledger reference for each submission.

### Evidence ledger requirement

Every signal submitted must cite the ledger where the underlying on-chain event occurred. This allows anyone to independently verify a signal by inspecting that ledger's transaction history. A signal without a verifiable evidence_ledger will be rejected by the `SignalAggregator`.

---

## Writing the SDK

The Vela.js SDK lives in `/sdk` and is written in TypeScript with strict mode.

### Conventions

- `camelCase` for functions and variables, `PascalCase` for classes and types
- All public methods and types must have JSDoc comments
- Explicit return types on all public API surfaces
- No `any` — use `unknown` and narrow explicitly

### Building

```bash
cd sdk
pnpm install
pnpm build
pnpm test
```

---

## Testing

### Contract tests

Live in each contract's `tests/` directory. Use the Soroban test environment.

Every PR touching contract logic must include:

- Happy path test
- All relevant `VelaError` variants
- Edge cases (zero signals, max values, boundary scores)

For reporter contracts, additionally include:

- Test that duplicate evidence_ledger submissions are rejected
- Test that out-of-range signal values are rejected
- Test that unregistered reporter calls are rejected

```bash
cargo test -p signal-aggregator
cargo test -p score-engine
cargo test -p score-token
```

### Scoring model tests

Any change to `model.rs` must include:

- Unit tests asserting exact score outputs for known input vectors
- Boundary tests (all signals at 0, all signals at 1, each signal at max with others at 0)
- Regression tests verifying existing passing test vectors still produce the same score

This prevents accidental scoring drift that would break lender integrations.

### Integration tests

```bash
cargo test --test integration
```

Required for changes to `SignalAggregator`, `ScoreEngine`, or any reporter contract.

---

## Pull request process

1. `cargo test` and `pnpm test` must pass before opening a PR
2. All PRs require one approving review from a maintainer
3. PRs touching `model.rs` (scoring weights/logic) require two approving reviews and an open governance discussion
4. PRs adding new reporters require one approving review and a governance registration proposal
5. Maintainers squash-merge approved PRs into `main`

### PR checklist

- [ ] Code compiles without warnings
- [ ] All tests pass
- [ ] New tests added for new behaviour
- [ ] Doc comments updated for changed public API
- [ ] `MODEL_VERSION` bumped if scoring logic changed
- [ ] `REPORTER_INTERFACE_VERSION` bumped if reporter interface changed
- [ ] PR description links the issue it resolves

---

*Built on [Stellar](https://stellar.org) and [Soroban](https://soroban.stellar.org).*
