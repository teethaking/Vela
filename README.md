# Vela

**Portable on-chain credit reputation protocol on Stellar.**

Vela builds a verifiable, non-custodial credit score from real on-chain behaviour — no self-reported data, no centralised bureau, no gatekeepers. Wallets accumulate reputation by transacting through Stellar anchors, repaying on-chain loans, maintaining consistent account activity, and participating in the Stellar DEX. The score is stored as a non-transferable Soulbound Token (SBT) on Soroban, queryable by any lender permissionlessly.

---

## Table of contents

- [Why Vela](#why-vela)
- [How it works](#how-it-works)
- [Signal sources](#signal-sources)
- [Scoring model](#scoring-model)
- [Architecture](#architecture)
- [Contract interface](#contract-interface)
- [The ScoreToken SBT](#the-scoretoken-sbt)
- [Lender integration](#lender-integration)
- [SDK — Vela.js](#sdk--velajs)
- [Fee structure](#fee-structure)
- [Security model](#security-model)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why Vela

1.4 billion people are financially invisible — not because they are untrustworthy, but because their trustworthiness exists outside the systems that banks look at. A market vendor in Lagos who transacts through a Stellar anchor every week, repays small on-chain loans on time, and maintains consistent account activity for two years is a demonstrably reliable financial actor. Today, no lender can see that.

Traditional credit bureaus have three structural problems that make them unfit for this population: they require bank accounts to generate history, they are centralised custodians of sensitive personal data, and they are geographically siloed — a score in one country is useless in another.

Vela solves all three. It reads exclusively from public on-chain Stellar activity. It stores only a derived score, never raw transaction data. And because it lives on Stellar, it is portable to any lender, in any country, that can make a Soroban contract call.

---

## How it works

### 1. History accumulates passively

Every time a Stellar address transacts through an anchor, repays an on-chain loan, participates in the Stellar DEX, or demonstrates long-term consistent activity, a signal is submitted to the `SignalAggregator` contract by a registered reporter. No opt-in required — history builds automatically from public chain data.

### 2. Score is calculated on demand

Any party — the wallet owner, a lender, or anyone paying the Soroban fee — can call `recalculate_score(address)`. The `ScoreEngine` reads all registered signals for the address, applies the weighted scoring model, and updates the `ScoreToken` SBT. Score range: 300–850.

### 3. Score is stored as a non-transferable SBT

The `ScoreToken` is minted once per address on first score calculation. It cannot be transferred, sold, or delegated. It is permanently bound to the originating address. Lenders query it directly from the contract.

### 4. Lenders query permissionlessly

Any on-chain protocol can call `get_score(address)` and receive the current score, band, last recalculation ledger, and signal breakdown. No API key, no data broker, no fee beyond the Soroban resource cost.

### 5. Score compounds over time

Every new reporter that registers with the `SignalAggregator` enriches scores retroactively. Every new lender that accepts Vela scores increases the practical value of a high score. The network effect compounds in both directions.

---

## Signal sources

Vela accepts signals from registered reporter contracts. Each reporter is whitelisted by governance and assigned a signal category and weight.

### Category 1 — Anchor payment history (35%)

On-time transactions via registered Stellar anchors (M-PESA, bKash, Wave, Flutterwave, MoneyGram). Each on-time payment increments the consistency score. Late or failed payments decrement it. Source: `AnchorPaymentReporter` — reads anchor transaction event streams.

### Category 2 — Transaction consistency (20%)

Regular, recurring transaction activity across the Stellar network. Measured as active months divided by total months since account creation. Weekly transactors score higher than sporadic high-volume transactors. Source: `TxConsistencyReporter`.

### Category 3 — Anchor volume and diversity (20%)

Total value transacted through Stellar anchors over a rolling 12-month window, weighted for asset diversity. Using multiple anchor providers signals broader financial participation. Source: `AnchorVolumeReporter`.

### Category 4 — On-chain loan repayment (15%)

Verified repayments of loans issued by registered on-chain lending protocols on Soroban. Timely repayments score positively, defaults score negatively. Source: registered `LoanReporter` contracts from Soroban lending protocols.

### Category 5 — Wallet age and activity density (7%)

Account age (ledgers since first transaction) multiplied by activity density (active months / total months). A three-year-old wallet active 30 of 36 months scores significantly higher than a new wallet with sudden high volume. Source: `WalletAgeReporter`.

### Category 6 — DEX participation (3%)

Participation in the Stellar decentralised exchange — order placement, fills, and liquidity provision. Signals financial sophistication and network engagement. Source: `DexActivityReporter`.

---

## Scoring model

```
raw_score = (
  anchor_payment_history  × 0.35 +
  tx_consistency          × 0.20 +
  anchor_volume           × 0.20 +
  loan_repayment          × 0.15 +
  wallet_age_density      × 0.07 +
  dex_participation       × 0.03
)

final_score = 300 + round(raw_score × 550)
```

Each component is normalised to a 0–1 range before weighting. The final score maps to four bands:

| Band | Range | Access |
|---|---|---|
| Poor | 300–499 | No lender access |
| Fair | 500–649 | Microfinance loans up to $200 |
| Good | 650–749 | DeFi collateral loans up to $1,000 |
| Great | 750–850 | Anchor credit lines up to $5,000+ |

Scores decay slowly when a wallet becomes inactive — 1 point per 30 days of zero activity — to reflect that stale history is less predictive than recent history.

---

## Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                        Vela Protocol                           │
│                                                                │
│  ┌──────────────────┐    ┌─────────────────────────────────┐  │
│  │ SignalAggregator  │    │         ScoreEngine             │  │
│  │                  │    │                                 │  │
│  │  SignalRecord[]  │───►│  Weighted model · 300–850       │  │
│  │  Per-address     │    │  Normalisation · Decay          │  │
│  │  Per-category    │    │  Recalculate on demand          │  │
│  └──────────────────┘    └───────────────┬─────────────────┘  │
│         ▲                                │                    │
│         │ report_signal()                ▼                    │
│  ┌──────┴──────────────────┐   ┌──────────────────────────┐  │
│  │    Reporter Contracts   │   │   ScoreToken (SBT)       │  │
│  │                         │   │                          │  │
│  │  AnchorPaymentReporter  │   │  Non-transferable        │  │
│  │  TxConsistencyReporter  │   │  One per address         │  │
│  │  AnchorVolumeReporter   │   │  Updated in-place        │  │
│  │  LoanReporter           │   │  Queryable by anyone     │  │
│  │  WalletAgeReporter      │   └──────────────────────────┘  │
│  │  DexActivityReporter    │                                  │
│  └─────────────────────────┘                                  │
└────────────────────────────────────────────────────────────────┘
         ▲                                      ▼
   Stellar network                        Lender contracts
   (anchors, DEX, loans)                  (DeFi, MFIs, anchors)
```

### Core components

**`SignalAggregator`** — Receives and stores `SignalRecord` entries from registered reporter contracts. Indexed by `(address, category)`. Exposes read functions to `ScoreEngine`.

**`ScoreEngine`** — Reads signals, applies the weighted model, normalises components, applies decay, calculates the final score, and updates the `ScoreToken`. Permissionlessly callable.

**`ScoreToken`** — Non-transferable Soroban token contract. Stores `(address → ScoreRecord)` mapping. No transfer function exists.

**Reporter contracts** — Thin adapter contracts that read Stellar on-chain events and translate them into `report_signal()` calls on the `SignalAggregator`. Each reporter is whitelisted by governance.

---

## Contract interface

### `report_signal`

Called by registered reporter contracts to submit a signal contribution.

```rust
pub fn report_signal(
    env: Env,
    reporter: Address,          // must be a registered reporter
    subject: Address,           // wallet being scored
    category: SignalCategory,
    value: i128,                // normalised 0–1_000_000
    evidence_ledger: u32,       // ledger where the underlying event occurred
) -> SignalId;
```

---

### `recalculate_score`

Called permissionlessly to trigger a score recalculation and SBT update.

```rust
pub fn recalculate_score(
    env: Env,
    subject: Address,
) -> ScoreRecord;
```

---

### `get_score`

View function for lenders and any external party.

```rust
pub fn get_score(
    env: Env,
    subject: Address,
) -> Option<ScoreRecord>;
```

---

### `get_signal_breakdown`

View function returning per-category signal values.

```rust
pub fn get_signal_breakdown(
    env: Env,
    subject: Address,
) -> SignalBreakdown;
```

---

### `register_reporter`

Admin-only function to whitelist a new reporter contract.

```rust
pub fn register_reporter(
    env: Env,
    admin: Address,
    reporter: Address,
    category: SignalCategory,
    name: String,
) -> ();
```

---

### Key types

```rust
pub struct ScoreRecord {
    pub subject: Address,
    pub score: u32,                   // 300–850
    pub band: ScoreBand,
    pub last_calculated_ledger: u32,
    pub signal_breakdown: SignalBreakdown,
    pub model_version: u32,
}

pub struct SignalBreakdown {
    pub anchor_payment_history: i128, // 0–1_000_000
    pub tx_consistency: i128,
    pub anchor_volume: i128,
    pub loan_repayment: i128,
    pub wallet_age_density: i128,
    pub dex_participation: i128,
}

pub enum ScoreBand {
    Poor,   // 300–499
    Fair,   // 500–649
    Good,   // 650–749
    Great,  // 750–850
}

pub enum SignalCategory {
    AnchorPaymentHistory,
    TxConsistency,
    AnchorVolume,
    LoanRepayment,
    WalletAgeDensity,
    DexParticipation,
}
```

---

## The ScoreToken SBT

The `ScoreToken` is a Soulbound Token — a non-transferable on-chain credential permanently bound to its originating address.

- Minted once per address on first `recalculate_score` call
- No transfer function — cannot be moved to another address
- Updated in-place on each recalculation — address history is preserved
- Score and breakdown are public — readable by anyone
- Underlying signal data stays in transaction history — the SBT stores only derived values

**Privacy:** Vela reads from public Stellar transaction history only. The score is a mathematical summary of already-public data. Users who want privacy should use Stellar's existing confidentiality features at the transaction level.

---

## Lender integration

### On-chain DeFi protocols

```rust
let score = vela_contract.get_score(&borrower)?;
require!(score.score >= MIN_SCORE, LenderError::InsufficientCredit);
require!(
    score.last_calculated_ledger >= env.ledger().sequence() - MAX_STALENESS,
    LenderError::StaleScore
);
```

### Microfinance institutions (API)

Query `GET /scores/:stellar_address` from the Vela backend — returns score, band, breakdown, and last updated timestamp. No API key required for read access.

### Anchor-based lenders (SDK)

```typescript
import { VelaClient } from '@vela-protocol/sdk';

const vela = new VelaClient({ network: 'mainnet' });
const score = await vela.getScore('GDKX…F7QA');

if (score.band === 'great') {
  await offerCreditLine(score.subject, 5000_00);
}
```

---

## SDK — Vela.js

### Installation

```bash
npm install @vela-protocol/sdk
```

### Get a score

```typescript
import { VelaClient } from '@vela-protocol/sdk';

const vela = new VelaClient({ network: 'mainnet' });
const score = await vela.getScore(address);

console.log(score.score);     // 718
console.log(score.band);      // "good"
console.log(score.breakdown); // per-category values
```

### Trigger recalculation

```typescript
const updated = await vela.recalculateScore({ wallet, subject: address });
```

---

## Fee structure

| Action | Who pays | Cost |
|---|---|---|
| `report_signal` | Reporter contract | ~0.001 XLM Soroban fee |
| `recalculate_score` | Caller (anyone) | ~0.003 XLM Soroban fee |
| `get_score` | Caller | ~0.0001 XLM (view function) |
| SBT mint (first score) | Caller | ~0.002 XLM one-time |

No protocol fee in v1. A governance-controlled fee may be introduced in v2.

---

## Security model

### What Vela can do

- Read publicly available on-chain Stellar transaction history
- Accept signal reports from whitelisted reporter contracts
- Calculate and store a derived score on a non-transferable token
- Allow anyone to read the score

### What Vela cannot do

- Accept signal reports from non-whitelisted reporters
- Modify scores without calling `recalculate_score`
- Transfer or move a ScoreToken to a different address
- Access or store any private data

### Threat model

**Reporter manipulation** — Mitigated by governance whitelisting, per-reporter signal caps, and the evidence_ledger requirement — every signal must cite a verifiable on-chain event.

**Sybil resistance** — Wallet age and activity density penalise freshly created addresses. A realistic high score requires months of consistent on-chain behaviour.

**Score gaming** — Multi-signal design prevents single-vector gaming. Inflating anchor payments is constrained by wallet age, DEX participation, and loan repayment signals that each require independent counterparties.

### Audit status

- [ ] Internal review — in progress
- [ ] External audit — planned pre-mainnet
- [ ] Bug bounty — planned post-audit

---

## Roadmap

### v0.1 — Testnet alpha
- [ ] `SignalAggregator`, `ScoreEngine`, `ScoreToken` contracts
- [ ] `AnchorPaymentReporter` and `WalletAgeReporter`
- [ ] Testnet deployment

### v0.2 — Full reporter set
- [ ] `TxConsistencyReporter`, `AnchorVolumeReporter`, `DexActivityReporter`
- [ ] `LoanReporter` standard for Soroban lending protocols
- [ ] Vela.js SDK and backend API

### v0.3 — Lender tooling + frontend
- [ ] Score dashboard (wallet owner view)
- [ ] Lender portal and query analytics
- [ ] Score history charting

### v1.0 — Mainnet
- [ ] External security audit
- [ ] Bug bounty program
- [ ] Mainnet deployment
- [ ] Anchor lender integrations

### v2.0 — Governance + expansion
- [ ] Protocol fee governance
- [ ] Score delegation (authorise a third party to query your score)
- [ ] Cross-chain score portability research

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for full guidelines.

```bash
git clone https://github.com/your-org/vela
cd vela
cargo build
cargo test
```

---

## License

MIT — see [LICENSE](./LICENSE).

---

*Built on [Stellar](https://stellar.org) and [Soroban](https://soroban.stellar.org).*
